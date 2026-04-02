# Event Planning System

A professional event planning and management solution for event planners to create, manage, and execute events with integrated registrations, ticketing, attendance tracking, notifications, and realtime updates.

This repository contains the complete system across two application layers, backed by PostgreSQL:

- **WEB (Frontend):** Nx + Next.js + TypeScript user interface
- **API (Backend):** ASP.NET modular monolith with semi-DDD (TBD: semi-DDD vs full DDD), Clean Architecture, CQRS, and MediatR
- **PostgreSQL (Database):** Schema-per-module isolation; module-owned EF Core migrations; Row-Level Security (RLS) planned (policy/versioning approach TBD)

Planned external services (TBD): Supabase Auth (Google SSO TBD) and Supabase Realtime.

## Status

**Stage:** Scaffold / MVP (TBD)

**Current scope:** The intended bounded contexts are Common, Users, Events, Registrations/Ticketing/Attendance, and Notifications. Refer to [TBD / In Progress](#tbd--in-progress) for implementation decisions that are not yet finalized.

**Current branching strategy:** Feature-based branching strategy.

## Table of Contents

- [Event Planning System](#event-planning-system)
- [Status](#status)
- [Table of Contents](#table-of-contents)
- [Key Capabilities](#key-capabilities)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Authorization (TBD)](#authorization-tbd)
- [WEB (Frontend)](#web-frontend)
- [API (Backend)](#api-backend)
- [Database (PostgreSQL)](#database-postgresql)
- [EF Core RLS Extension Design](#ef-core-rls-extension-design)
- [External Services (TBD)](#external-services-tbd)
- [Bounded Contexts / Modules](#bounded-contexts--modules)
- [Repository Structure](#repository-structure)
- [Local Development Strategy](#local-development-strategy)
- [Environment Variables](#environment-variables)
- [Commands (Cheat Sheet)](#commands-cheat-sheet)
- [Code Quality Gates](#code-quality-gates)
- [Testing](#testing)
- [TBD / In Progress](#tbd--in-progress)

## Key Capabilities

Current and intended capabilities mapped to bounded contexts:

- **Users / Event Planners**
  - Registration and profile management (roles TBD)
  - Authentication via platform auth services

- **Events**
  - Create, read, update, and delete events
  - Event scheduling and timeline management
  - Event status tracking
  - Realtime event manager dashboard (TBD; intended for live event, registration, and status monitoring)

- **Registrations / Ticketing / Attendance**
  - Ticket creation and inventory management
  - Event registration and attendee management
  - Attendance tracking
  - Capacity management and ticket sales rules (payments TBD)

- **Notifications**
  - Event notifications to attendees (channels/providers TBD)
  - Realtime updates (TBD; planned via Supabase Realtime)

- **Authorization (TBD)**
  - Role-based access control (RBAC) managed by the Auth service (Supabase Auth; Google SSO TBD)
  - Endpoint-level authorization for API endpoint access (role requirements; TBD)
  - Database-level authorization via PostgreSQL RLS (ownership + read/write rules; TBD)
  - Optional use-case-level authorization for resource-specific rules (may require configuration/matrix; TBD)

**Out of scope or deferred:** Payments/subscriptions (TBD), advanced analytics, multi-tenancy (TBD), mobile apps (TBD).

## Tech Stack

### WEB (Frontend)

- Nx monorepo workspace
- Next.js
- TypeScript
- Tailwind CSS
- shadcn/ui
- Lucide Icons
- RTK Query
- Realtime subscription support (TBD; planned via Supabase Realtime)

**Version targets (TBD):** Node.js LTS, Nx version, and Next.js version still need to be confirmed.

### API (Backend)

- ASP.NET (C#) modular monolith
- Semi-DDD modular monolith direction (TBD: semi-DDD vs full DDD)
- Clean Architecture and SOLID principles
- MediatR for CQRS handlers
- CQRS pattern for commands and queries
- StyleCop for code style enforcement
- Structured logging (implementation TBD)

**Version targets (TBD):** .NET SDK version still needs to be confirmed.

### Database Stack (PostgreSQL)

- PostgreSQL
- Entity Framework Core + Npgsql
- See [Database (PostgreSQL)](#database-postgresql) for connectivity, schema isolation, RLS, and migration/versioning concerns.

### External Integrations (TBD)

- Supabase Auth (planned; Google SSO integration TBD)
- Supabase Realtime (planned; integration details TBD)

## Architecture Overview

### Coding Principles

- Keep business rules inside API (Backend) Domain and Application layers.
- Keep WEB (Frontend) focused on composition, presentation, and client-side interaction.
- Use schema-level isolation to enforce clear module boundaries in PostgreSQL.
- Standardize authorization across layers (endpoint roles + database RLS; optional use-case checks if needed) (TBD).
- Roles are managed by the Auth service (planned: Supabase Auth; Google SSO TBD); claim/role mapping into the API is TBD.
- Avoid hardcoding role strings across the codebase; centralize role/policy contracts (exact location TBD).
- If/when realtime is introduced, align publication/subscription rules with the same authorization model after the RLS authorization approach is researched (TBD).
- Keep external integrations (e.g., Supabase Auth/Realtime) behind clear interfaces; provider-specific details remain isolated (TBD).
- Keep secrets out of source control and out of browser bundles.

### System Design Architecture

The system is organized as an API layer plus DB-driven realtime delivery (planned; TBD):

1. WEB calls API for commands and queries.
2. API executes business workflows, enforces authorization (endpoint roles + optional use-case checks; TBD), and persists using the centralized Database (PostgreSQL) model.
3. Database operations enforce module-level schema isolation and planned RLS controls (TBD/WIP).
4. Realtime delivery is planned as DB-driven realtime (target: Supabase Realtime) for event delivery (TBD).
5. Authorization semantics must be standardized so API decisions and DB-driven realtime visibility follow the same authorization model (TBD).

### Coding Patterns

#### WEB (Frontend)

- Feature-first UI organization inside Nx workspace.
- RTK Query for server state and caching.
- shadcn/ui primitives for reusable UI building blocks.
- Realtime subscriptions (TBD) consume API-authorized event streams and channels.
- Authorization-aware UI states should derive from API-exposed authorization context (roles/claims; TBD), not hardcoded role strings.

#### API (Backend)

- Structure includes a dedicated Common module (`API/src/common`) plus feature modules (Users, Events, Registrations, Notifications).
- Clean Architecture per module with Domain, Application, and Infrastructure boundaries.
- Use-case-driven Application layer where commands/queries, handlers, and validators are centralized per use case.
- MediatR request/handler flow for commands and queries.
- AutoMapper profiles defined under an Application `Profiles` directory.
- Database operations (ORM, connection model, schema isolation, RLS, and migration/versioning) are centralized under [Database (PostgreSQL)](#database-postgresql).
- Authorization behavior follows the model in [Authorization (TBD)](#authorization-tbd).
- Inter-module communication through service interfaces and contracts, not direct domain references, implemented via Common.Infrastructure cross-cutting components.
- API depends on capability interfaces; external provider adapters (e.g., Supabase Auth/Realtime) are implemented behind those interfaces (TBD).

### Logging Style (TBD)

- Logging library choice is not finalized.
- Preferred direction is structured logging with consistent event names and properties.
- Authorization failures and unexpected role/claim issues should be explicitly logged.
- Correlation IDs, request IDs, and module context should be documented once implementation is confirmed.
- Sensitive data must be redacted from logs.

## Authorization (TBD)

The system will use a role-based access control (RBAC) model managed by the Auth service (planned: Supabase Auth; Google SSO provider TBD).

This repository does **not** use a startup-loaded permission-matrix bootstrap model. Instead, authorization is applied at multiple layers (defense in depth) with final implementation details intentionally left as TBD until the Auth/Realtime integrations are finalized.

### 1) Endpoint-Level Authorization (API)

Primary purpose: control access to API endpoints based on caller roles.

- Endpoints define required roles (exact roles and naming conventions are TBD).
- Enforcement should live at the endpoint boundary using the platform’s authorization primitives (policies/attributes/middleware) rather than ad-hoc checks scattered through business logic.
- Where UI behavior depends on access, WEB should derive state from API-exposed authorization context (roles/claims; TBD) rather than duplicating rules.

### 2) Database-Level Authorization (PostgreSQL RLS)

Primary purpose: control row-level data access (ownership, read/write rules per table).

- Implemented using PostgreSQL Row-Level Security (RLS) policies.
- How identity/claims are propagated from API requests into PostgreSQL sessions, and how RLS policies are versioned/applied, is TBD and requires further research.
- RLS complements (not replaces) endpoint and use-case authorization; business invariants remain enforced in the Application/Domain layers.

### 3) Use-Case-Level Authorization (Application Layer; Optional)

Purpose: apply resource-specific authorization that cannot be expressed as a simple endpoint role check or a pure RLS predicate.

- Implemented within Application layer use cases/handlers so it stays testable and close to the workflow.
- May require role/resource configuration (a “permissions matrix”) if rules must be data-driven; this is TBD and likely deferred unless a concrete use case requires it.

## WEB (Frontend)

The WEB (Frontend) layer is the user experience for professional event planners.

### Responsibilities

- Render event planning workflows.
- Manage feature composition and client-side routing.
- Call API (Backend) endpoints through RTK Query.
- Subscribe to realtime channels for live updates (TBD; planned Supabase Realtime).
- Respect API-exposed authorization context/contracts (roles/claims; TBD) in UI behavior.

### Suggested workspace areas

- `WEB/apps/web` for the main Next.js application.
- `WEB/libs/ui` for shared UI primitives and wrappers.
- `WEB/libs/data-access` for API slices, query hooks, and realtime adapters.
- `WEB/libs/shared` for shared types, enums, and utilities.

### Coding expectations

- Keep page-level code thin.
- Put reusable presentation logic into shared UI components.
- Keep data fetching in dedicated data-access code.
- Keep authorization checks centralized and reusable.
- Avoid duplicating authorization rules in multiple UI surfaces.

## API (Backend)

The API (Backend) layer is the modular monolith that owns business rules, authorization enforcement, and persistence orchestration. It is intended to follow a semi-DDD approach, with the final choice between semi-DDD and fully fledged DDD still TBD.

### Responsibilities

- Validate and execute application workflows.
- Own domain logic for all bounded contexts.
- Enforce endpoint-level authorization (roles) for protected API endpoints (TBD).
- Apply use-case-level authorization when workflows require resource-specific checks (TBD).
- Coordinate persistence workflows using the centralized Database (PostgreSQL) model.
- Integrate with the Auth service for authentication and role/claim evaluation (Supabase Auth; Google SSO TBD).
- Expose stable contracts to WEB (Frontend) and external service integrations (TBD).

### Suggested workspace areas

- `API/src/Api` for host and composition root.
- `API/src/BuildingBlocks` for shared abstractions.
- `API/src/common` for Common module cross-cutting concerns and shared authorization contracts/helpers (TBD).
- `API/src/Modules/*` for feature bounded contexts and their internal layers.
- `API/tests` for test implementation (exact structure TBD).

## Database (PostgreSQL)

This section centralizes database-specific concerns, including ORM/data access, connection method, schema isolation, RLS policies, and migration/versioning strategy.

### Responsibilities

- Provide durable storage for module data and shared Common schema data.
- Enforce module boundaries through schema isolation.
- Support authorization-aware access patterns aligned with API authorization contracts.
- Define migration and schema change ownership/lifecycle.
- Define Row-Level Security (RLS) policy strategy (TBD/WIP).

### ORM and Connectivity Model

- ORM/data access: Entity Framework Core with Npgsql.
- Runtime connection method: standard PostgreSQL connection string (`ConnectionStrings__Main`).
- Runtime access path: API process owns runtime database reads/writes.
- Final conventions for pooling/retry/timeout tuning are TBD/WIP.

### Schema Isolation Model

- Each feature module owns a dedicated PostgreSQL schema.
- Common module may own its own schema for shared kernel tables (TBD).
- Cross-schema access should be minimized and mediated through module contracts.
- Final schema naming conventions are TBD.

### RLS Policies and Versioning (TBD/WIP)

- PostgreSQL RLS will be used as an additional enforcement layer for row-level data access.
- Exact policy mapping to application identity/claims is TBD.
- Current direction is code-first EF migrations with explicit policy operations; see [EF Core RLS Extension Design](docs/EF-CORE-RLS-EXTENSION-DESIGN.md).
- Final rollout and module adoption sequencing remain TBD.
- Policy rollout, verification, and rollback process is WIP.

### Migration and Schema Versioning Model

- Migrations are owned and executed by API modules.
- Each module maintains its own DbContext and migration stream.
- Common module migrations are owned under `API/src/common`.
- Cross-module schema changes require coordinated review (process TBD).
- Release-time migration execution model (pipeline/manual/startup gate) is TBD/WIP.

### Local Database Operations

- Each developer provisions a local PostgreSQL instance.
- Configure `ConnectionStrings__Main` for local runtime.
- Apply module-specific migrations before API startup.
- Validate RLS policy behavior after schema updates if/when RLS is enabled (TBD).
- Use command references under [Commands (Cheat Sheet)](#commands-cheat-sheet).

## EF Core RLS Extension Design

The canonical blueprint for code-first RLS policy management in EF Core is documented in:

- [EF Core RLS Extension Design](docs/EF-CORE-RLS-EXTENSION-DESIGN.md)

This design aligns with the architecture in this repository:

- Common module placement for reusable, cross-cutting persistence concerns.
- Strongly typed policy definitions modeled as EF metadata.
- Snapshot-aware migration diffing with explicit create/alter/drop policy operations.
- Provider-aware execution with PostgreSQL as the initial target.

## External Services (TBD)

Supabase Auth and Supabase Realtime are planned integrations. Exact implementation details (integration approach, hosting model, and local development provisioning) are TBD.

### Supabase Auth (TBD)

- Token validation and claims mapping strategy (TBD).
- Role model and assignment strategy (RBAC), and how roles map to endpoint authorization requirements (TBD).
- Google SSO integration approach (TBD).

### Supabase Realtime (TBD)

- Channel naming and event contract conventions (TBD).
- Publication/subscription authorization alignment with the application authorization model (endpoint roles + RLS + optional use-case checks) (TBD; depends on the RLS authorization research outcome).

## Bounded Contexts / Modules

### Common Module

- **Responsibility:** Shared kernel and cross-cutting conventions.
- **Location:** `EventPlanningSystem/API/src/common`.
- **Authorization responsibility:** Define shared authorization contracts (roles/claims) and cross-cutting authorization helpers/policies (TBD).
- **Data ownership:** Common schema tables for shared kernel data (if any) (TBD).
- **Runtime responsibility:** No permission-matrix bootstrap; authorization is driven by Auth roles + RLS + (optional) use-case rules (TBD).
- **Cross-cutting responsibility:** Common.Infrastructure is the only place for cross-cutting implementations used by inter-module communication.
- **Roles and permissions:** Role catalog and endpoint mapping conventions are TBD and will be managed via the Auth service.

### Users Module

- **Responsibility:** Event planner user management, profile, and roles.
- **Entities/Aggregates:** User, RoleAssignment, Profile.
- **Main Commands:** CreateUserCommand, UpdateUserCommand, UpdateRoleCommand (if applicable).
- **Main Queries:** GetUserQuery, GetUserProfileQuery, ListUsersQuery (if applicable).
- **Schema ownership:** Dedicated users schema.

### Events Module

- **Responsibility:** Event lifecycle management, scheduling, and status.
- **Entities/Aggregates:** Event, EventSchedule, EventStatus.
- **Main Commands:** CreateEventCommand, UpdateEventCommand, PublishEventCommand, CancelEventCommand.
- **Main Queries:** GetEventQuery, ListEventsQuery, GetEventDetailsQuery.
- **Schema ownership:** Dedicated events schema.

### Registrations Module

- **Responsibility:** Ticket inventory, attendee registration, capacity management, and attendance tracking.
- **Entities/Aggregates:** Ticket, TicketType, Registration, Attendee, Attendance.
- **Main Commands:** CreateTicketCommand, UpdateTicketCommand, RegisterAttendeeCommand, RecordAttendanceCommand.
- **Main Queries:** GetTicketQuery, ListTicketsQuery, GetRegistrationQuery, ListRegistrationsQuery.
- **Schema ownership:** Dedicated registrations schema.

### Notifications Module

- **Responsibility:** Notification orchestration and delivery integration.
- **Entities/Aggregates:** Notification, NotificationTemplate, NotificationChannel.
- **Main Commands:** SendNotificationCommand, CreateNotificationTemplateCommand.
- **Main Queries:** GetNotificationQuery, ListNotificationsQuery.
- **Schema ownership:** Dedicated notifications schema.

### Inter-Module Communication

- Modules communicate only through interfaces and contracts.
- Domain entities are not shared directly across module boundaries.
- Each module owns persistence in its own schema boundary.
- Shared authorization contracts (roles/claims/policies; TBD) are consumed as a shared contract, not duplicated logic.
- Cross-cutting communication implementation must live in `common/Infrastructure`; feature modules consume interfaces and must not implement shared cross-cutting communication directly.

## Repository Structure

### Proposed Directory Tree

```text
EventPlanningSystem/
├── .github/
│   └── workflows/                      (CI/CD pipelines; TBD)
├── WEB/                                (WEB (Frontend) layer)
│   ├── apps/
│   │   └── web/                        (main Next.js application)
│   └── libs/
│       ├── ui/                         (shared UI primitives)
│       ├── data-access/                (API calls and realtime adapters)
│       └── shared/                     (shared types, utilities, constants)
├── API/                                (API (Backend) layer)
│   ├── src/
│   │   ├── Api/                        (host, composition root)
│   │   ├── BuildingBlocks/             (cross-cutting abstractions)
│   │   ├── common/                     (Common module: shared kernel and cross-cutting conventions)
│   │   └── Modules/
│   │       ├── Users/
│   │       ├── Events/
│   │       ├── Registrations/
│   │       └── Notifications/
│   └── tests/                          (TBD: final test structure)
├── docs/
│   ├── architecture/                   (deeper architecture docs)
│   └── development/                    (setup and configuration guides)
├── tools/                              (workspace scripts and helpers)
├── README.md                           (this file)
└── .gitignore
```

### Proposed Module Internal Structure (Clean Architecture)

Each feature module under `API/src/Modules/*` should follow this internal structure:

```text
<ModuleName>/
├── Presentation/
│   ├── Endpoints/
│   ├── Contracts/
│   └── Mapping/
├── Application/
│   ├── UseCases/
│   │   └── <UseCaseName>/
│   │       ├── Command.cs or Query.cs
│   │       ├── Handler.cs
│   │       └── Validator.cs
│   ├── Profiles/
│   └── Services/
├── Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Aggregates/
│   ├── DomainEvents/
│   └── Repositories/
└── Infrastructure/
  ├── Persistence/
  │   ├── DbContext/
  │   ├── Configurations/
  │   └── Migrations/
  ├── Integrations/
  └── Repositories/
```

Common module location and layering:

```text
API/src/common/
├── Presentation/
├── Application/
├── Domain/
└── Infrastructure/
```

Application layer notes:

- Use case folders can group logic around one or more relevant domain entities and are not required to map 1:1 to a single entity.
- AutoMapper profiles are defined in `Application/Profiles` and can aggregate mappings across related use cases.

Common module cross-cutting note:

- Cross-cutting concerns are implemented only in `common/Infrastructure`.
- This layer provides the shared implementations that enable inter-module communication via interfaces/contracts.

### Directory Purpose & What Belongs Where

| Directory | Purpose | Contains | Layer |
|-----------|---------|----------|-------|
| `WEB/apps/web` | WEB application | Pages, routes, layouts, feature components | WEB (Frontend) |
| `WEB/libs/ui` | Shared UI components | shadcn/ui wrappers, icon patterns, form UI | WEB (Frontend) |
| `WEB/libs/data-access` | API and realtime integration | RTK Query slices, realtime adapters, authorization-aware clients | WEB (Frontend) |
| `WEB/libs/shared` | Shared types and utilities | DTOs, enums, constants, helpers | Shared contracts |
| `API/src/Api` | API host | Startup, DI, middleware, module registration, auth/authorization wiring (TBD) | API (Backend) |
| `API/src/common` | Common module | Shared kernel, authorization contracts/helpers (TBD) | API (Backend) |
| `API/src/Modules/Users` | User planner management | User domain, profile management, role logic | API (Backend) |
| `API/src/Modules/Events` | Event lifecycle | Event domain, scheduling logic, status management | API (Backend) |
| `API/src/Modules/Registrations` | Ticketing and attendance | Ticket, registration, attendance domains | API (Backend) |
| `API/src/Modules/Notifications` | Notification delivery | Notification domain, template management, delivery logic | API (Backend) |
| `API/tests` | Test implementation area | Final test structure and layout are TBD | API (Backend) |

## Local Development Strategy

This section combines setup flow and local run flow into one place.

### Branching Strategy

- **Current strategy:** feature-based branching strategy.
- **Default branch shape:** `feature/<area>-<short-description>` (convention to be finalized).
- **TBD:** release branch policy, hotfix policy, merge strategy, and version tagging strategy.

### Prerequisites

Before setup, confirm:

- Node.js (version TBD; recommend LTS)
- Package manager (TBD: pnpm, npm, or yarn)
- Nx CLI
- .NET SDK (version TBD; recommend LTS)
- Local PostgreSQL instance per developer
- (TBD) Supabase Auth project access (Google SSO TBD)
- (TBD) Supabase Realtime for realtime flows

### First-Time Setup

1. Clone the repository.
2. Install WEB (Frontend) dependencies using the selected package manager.
3. Restore API (Backend) dependencies with .NET CLI.
4. Provision local PostgreSQL for your developer environment.
5. (TBD) Provision Supabase Realtime for developer realtime flows.
6. Configure environment variables (PostgreSQL connection string; auth/realtime endpoints TBD).
7. Apply module-specific migrations (one DbContext per module).
8. Start API and validate auth/authorization wiring (TBD).
9. Start WEB and validate API + planned auth/realtime integration (TBD).

### Developer Environment Model

- Each developer runs their own local database.
- Supabase Auth and Supabase Realtime are planned; the exact local development provisioning model is TBD.
- Containerized local development may be introduced later for easier onboarding and environment consistency (TBD).

### Database Setup (Connection String Driven)

- Follow the centralized model in [Database (PostgreSQL)](#database-postgresql) for ORM, connectivity, schema isolation, RLS, and migration/versioning guidance.
- Configure `ConnectionStrings__Main` for local PostgreSQL access.
- Execute module migrations from API module projects/DbContexts before startup.

### Start API (Backend)

```bash
cd API
dotnet restore
dotnet run
```

On startup, API should:

- connect to PostgreSQL using configured connection string,
- initialize authentication/authorization components (token validation + role/claim mapping; TBD),
- make request identity/roles available to the application and (if needed) to database sessions for RLS evaluation (TBD).

### Start WEB (Frontend)

```bash
cd WEB
pnpm nx serve web
```

If repository standard uses a different package manager, replace command accordingly.

### Expected URLs

| Service | Expected URL | Notes |
|-------|--------------|-------|
| WEB (Frontend) | `http://localhost:3000` | Next.js dev server (TBD) |
| API (Backend) | `http://localhost:5000` | ASP.NET dev server (TBD) |
| Auth (Supabase; TBD) | `TBD` | Hosted/shared model TBD |
| Realtime (Supabase; TBD) | `TBD` | Per-developer vs shared TBD |

### Setup Notes

- Keep local configuration out of source control.
- Prefer one canonical startup flow for onboarding.
- Validate auth/authorization startup behavior in API logs before testing secured flows (TBD).

## Environment Variables

Environment variables are grouped by runtime area to keep secrets secure and keep local development repeatable.

### WEB (Frontend) Variables

Place in `.env.local` or equivalent local environment file.

| Variable | Purpose | Visibility |
|----------|---------|------------|
| `NEXT_PUBLIC_API_URL` | API (Backend) base URL | Public |
| `NEXT_PUBLIC_RT_URL` | Realtime endpoint URL | Public |
| `NEXT_PUBLIC_AUTH_URL` | Auth endpoint URL | Public |

### API (Backend) Variables and Configuration

- `appsettings.json` for committed base configuration.
- `appsettings.Development.json` for developer overrides.
- Environment variables for machine-specific values.
- User Secrets for local sensitive values.

| Category | Names to document | Notes |
|----------|-------------------|-------|
| Database | `ConnectionStrings__Main` | Primary PostgreSQL connection string |
| Database schemas | `Database__Schemas__Common`, `Database__Schemas__Users`, `Database__Schemas__Events`, `Database__Schemas__Registrations`, `Database__Schemas__Notifications` | Schema boundary config |
| Realtime | `Realtime__Url`, `Realtime__ServiceKey` | Server-side realtime integration settings (TBD) |
| Auth | `Auth__Issuer`, `Auth__Audience`, `Auth__SigningKey` | Supabase Auth integration settings (TBD) |
| Authorization | TBD | Endpoint role policies and use-case-level authorization configuration (if needed) (TBD) |
| Logging | `Logging__LogLevel__Default` | Default log level |

### Auth and Realtime (TBD)

- Supabase Auth integration is planned; issuer/audience/keys, claims mapping, and role model are TBD.
- Supabase Realtime integration is planned; endpoints/keys and channel conventions are TBD.
- Local development provisioning model for Auth/Realtime is TBD.
- API module migrations remain versioned and executed from API module DbContexts.

### Secrets Handling

- Never commit connection strings, service keys, or secret tokens.
- Browser-exposed values must be explicitly safe for public exposure.
- Server-only values remain in API runtime configuration and secure secret stores.

## Commands (Cheat Sheet)

### WEB (Frontend)

| Command | Purpose |
|---------|---------|
| `pnpm nx serve web` | Start Next.js development server |
| `pnpm nx build web` | Build WEB application |
| `pnpm nx test web` | Run WEB tests |
| `pnpm nx lint web` | Lint WEB code |
| `pnpm nx typecheck web` | Type check WEB code |
| `pnpm nx format:check web` | Check formatting |

### API (Backend)

| Command | Purpose |
|---------|---------|
| `dotnet restore` | Restore NuGet packages |
| `dotnet build` | Build API solution |
| `dotnet run` | Start API host |
| `dotnet test` | Run API tests |
| `dotnet ef migrations add <Name> --context <ModuleDbContext>` | Create migration for a specific module |
| `dotnet ef database update --context <ModuleDbContext>` | Apply migrations for a specific module |

### Database Utilities

| Command | Purpose |
|---------|---------|
| `dotnet ef database update --context <ModuleDbContext>` | Apply schema changes for one module DbContext |
| `dotnet ef migrations script` | Generate SQL script for review/deployment |
| `psql <connection-string>` | Manual validation of schemas/tables (if psql is installed) |

### All Checks

```bash
cd WEB
pnpm nx lint web
pnpm nx typecheck web
pnpm nx test web
pnpm nx build web

cd ../API
dotnet build
dotnet test
cd ..
```

## Code Quality Gates

### WEB (Frontend)

| Check | Tool | Target | Pass Criteria |
|-------|------|--------|---------------|
| Linting | ESLint | `pnpm nx lint web` | Zero errors |
| Type checking | TypeScript | `pnpm nx typecheck web` | Zero errors |
| Testing | Test runner TBD | `pnpm nx test web` | All tests pass |
| Formatting | Prettier or equivalent | `pnpm nx format:check web` | Code matches format |
| Build | Next.js | `pnpm nx build web` | Builds successfully |

### API (Backend)

| Check | Tool | Target | Pass Criteria |
|-------|------|--------|---------------|
| Build | .NET compiler | `dotnet build` | Zero compiler errors |
| Testing | Test framework TBD | `dotnet test` | All tests pass |
| StyleCop | StyleCop analyzers | `dotnet build` | Zero style violations, policy TBD |
| Code analysis | Roslyn analyzers | `dotnet build` | Zero analysis errors |
| Schema isolation validation | Integration tests + DB checks | Test suite + schema assertions | No cross-module schema leakage |
| Authorization enforcement validation | Integration tests + security checks (TBD) | Protected endpoints + RLS behavior (TBD) | Unauthorized access is denied; authorized access is allowed |

### PR Minimum Bar

- WEB (Frontend) lint, typecheck, test, and build pass.
- API (Backend) build and tests pass.
- StyleCop and analyzer results are clean according to agreed policy.
- Schema isolation and authorization checks pass (endpoint roles + RLS; TBD).
- No secrets or credentials are committed.

## Testing

Testing structure and strategy are intentionally deferred for now.

- `API/tests` exists as a placeholder and final structure is TBD.
- Module-level test layout (unit/integration) is TBD.
- WEB test layout and framework finalization are TBD.
- Coverage thresholds and reporting enforcement are TBD.

## TBD / In Progress

This section centralizes unresolved decisions, missing information, and items still under implementation.

### Setup and Tooling

- [ ] Node.js version and package manager choice.
- [ ] Nx version and workspace command conventions.
- [ ] .NET SDK version and whether `global.json` is used.
- [ ] Canonical local PostgreSQL provisioning approach.
- [ ] Exact local ports for WEB and API.
- [ ] Final startup verification checklist.

### Architecture and Backend

- [ ] Final schema names and naming conventions for all modules.
- [ ] Migration ownership model (EF-only, SQL-only, or combined workflow).
- [ ] Role model and how roles are represented in tokens/claims (TBD).
- [ ] Endpoint-level authorization conventions (roles, policies, naming, and ownership) (TBD).
- [ ] Use-case-level authorization approach and whether role/resource configuration is needed (permissions matrix TBD).
- [ ] Realtime authorization alignment mechanism details.
- [ ] PostgreSQL Row-Level Security (RLS) policy design, plus how policies are versioned/applied.
- [ ] Logging library choice and correlation ID strategy.

### Product Scope

- [ ] Stage or maturity label.
- [ ] Auth roles and access rules.
- [ ] Multi-tenancy requirement.
- [ ] Payments scope for registrations and ticketing.
- [ ] Notification channels and providers.
- [ ] Additional bounded contexts beyond current scope.

### WEB (Frontend)

- [ ] Next.js version and routing approach.
- [ ] RTK Query organization strategy.
- [ ] Realtime subscription pattern finalization.
- [ ] API versioning strategy.
- [ ] Authorization-driven UI behavior guidelines.

### Code Quality and CI/CD

- [ ] CI system choice.
- [ ] Required merge checks.
- [ ] Coverage thresholds.
- [ ] Frontend test framework finalization.
- [ ] Backend test framework finalization.
- [ ] StyleCop enforcement policy.

### Deployment and Environments

- [ ] WEB hosting target.
- [ ] API hosting target.
- [ ] PostgreSQL environment strategy.
- [ ] Realtime/Auth environment strategy.
- [ ] Backup and disaster recovery plan.
- [ ] Secrets management approach.
- [ ] Environment templates for team standardization.

### Documentation

- [x] EF Core RLS extension blueprint added at [docs/EF-CORE-RLS-EXTENSION-DESIGN.md](docs/EF-CORE-RLS-EXTENSION-DESIGN.md).
- [ ] Need for deeper docs under `docs/architecture`.
- [ ] Need for development playbooks under `docs/development`.
- [ ] Need for architecture decision records.
