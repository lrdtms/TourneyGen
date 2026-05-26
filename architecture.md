# TourneyGen — Architecture Document

**Version:** 1.0  
**Date:** 2026-05-26  
**Status:** Proposed  

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Tech Stack Decisions](#2-tech-stack-decisions)
3. [Blazor Render Mode Strategy](#3-blazor-render-mode-strategy)
4. [Project Structure](#4-project-structure)
5. [Data Model](#5-data-model)
6. [API / Service Layer](#6-api--service-layer)
7. [Authentication Architecture](#7-authentication-architecture)
8. [Guest Session Strategy](#8-guest-session-strategy)
9. [Sharing Architecture](#9-sharing-architecture)
10. [PDF Export](#10-pdf-export)
11. [Key Non-Functional Requirements](#11-key-non-functional-requirements)
12. [Future MAUI Hybrid Path](#12-future-maui-hybrid-path)
13. [Deployment Architecture](#13-deployment-architecture)

---

## 1. System Overview

TourneyGen is a single-server, server-rendered web application. It is not a distributed system. There is one deployable unit (the ASP.NET Core/Blazor application) backed by a MySQL database and Linode Object Storage. This is deliberate: the scale and team size do not justify distribution, and a modular monolith delivers the same logical boundaries with far less operational complexity. See *Building Microservices* (Newman) ch. 1: "Don't start with microservices."

### C4 Context Diagram

```
+-----------------------------------------------+
|                 <<Internet>>                  |
|                                               |
|  +------------+      +--------------------+  |
|  |  Organiser  |      |   Viewer/Guest     |  |
|  | (Logged-in) |      | (Browser only)     |  |
|  +------+------+      +--------+-----------+  |
|         |  HTTPS                | HTTPS        |
|         v                       v              |
|  +------+-------------------------+            |
|  |        TourneyGen Web App      |            |
|  |   (ASP.NET Core + Blazor)      |            |
|  |   Hosted on Linode VPS         |            |
|  +------+---------+----------+---+            |
|         |         |          |                 |
|         | SQL     | SMTP     | S3 API          |
|         v         v          v                 |
|  +------+--+ +-------+ +----------+           |
|  |  MySQL  | | Gmail | |  Linode  |           |
|  | (Docker)| | SMTP  | |  Object  |           |
|  +---------+ +-------+ | Storage  |           |
|                        +----------+           |
+-----------------------------------------------+
```

### Major Components (C4 Container Level)

| Container | Technology | Responsibility |
|---|---|---|
| **Blazor Web App** | ASP.NET Core 8 + Blazor | All UI and application logic. Renders pages server-side by default; elevates to InteractiveServer for stateful bracket/league management UI. |
| **Application Services** | C# classes, injected into Blazor | Encapsulate all business logic: tournament bracket computation, league standings, active-event cap enforcement, PDF orchestration. |
| **MySQL Database** | MySQL 8, Docker container | Persistent state for all organiser-owned data: users, tournaments, leagues, participants, matches, results, shared links. |
| **Object Storage** | Linode Object Storage (S3-compatible) | PDF binary storage. Not used for any application state. |
| **Gmail SMTP** | External service | Transactional email: confirmation, password reset. |

---

## 2. Tech Stack Decisions

### 2.1 ASP.NET Core + Blazor Web App

**Decision:** Use the unified Blazor Web App project template (introduced in .NET 8), not separate Blazor Server or Blazor WASM projects.

**Justification:**
- The Blazor Web App template is the .NET 8+ standard and supports all render modes (SSR, InteractiveServer, InteractiveWASM, InteractiveAuto) within a single project without duplication.
- It aligns directly with the future MAUI Hybrid path: the `RazorClassLibrary` that will house shared components can be consumed by both the Web App and the MAUI Shell without rework.
- The application is organiser-facing CRUD with rich stateful UI (drag-to-reseed brackets, live result entry). This is exactly the Blazor sweet spot; a JS SPA framework would require duplicating business logic in TypeScript.

**Tradeoff accepted:** Blazor's runtime on the server carries slightly higher memory per-connection than a pure static API + SPA architecture. At projected single-server scale with a 5-event-per-user cap, this is not material.

### 2.2 Entity Framework Core + Pomelo MySQL

**Decision:** Use EF Core 8 with the Pomelo.EntityFrameworkCore.MySql provider (not the Oracle MySql.EntityFrameworkCore provider).

**Justification:**
- Pomelo is the community-standard .NET MySQL provider; it is actively maintained, has wider .NET 8 feature coverage than Oracle's official provider, and has documented support for EF Core migrations.
- EF Core Code-First migrations are the right fit for a greenfield project with a single developer or small team — schema changes stay version-controlled alongside code.
- The data model has clear relational structure (tournaments own participants, participants are in matches, matches belong to rounds). An ORM is appropriate; there is no query complexity that would demand raw SQL or a different data access strategy.

**Tradeoff accepted:** EF Core adds a migration discipline cost. Direct SQL would be faster to prototype but harder to evolve safely. EF Core wins on maintainability.

### 2.3 ASP.NET Core Identity

**Decision:** Use ASP.NET Core Identity as the membership system for email/password and OAuth.

**Justification:**
- Identity is already designed to integrate with EF Core, which means the `ApplicationUser` table lives in the same database and migrations as domain data. No separate identity store to manage.
- It provides the complete email confirmation + password reset flow out of the box with Gmail SMTP wired to `IEmailSender`.
- Google OAuth is supported via `Microsoft.AspNetCore.Authentication.Google` middleware — two NuGet packages, no custom OAuth implementation.

**Tradeoff accepted:** Identity's schema (`AspNetUsers`, `AspNetRoles`, etc.) is heavier than a hand-rolled users table. The overhead is negligible for this scale and the saved implementation time is significant.

### 2.4 Linode VPS + MySQL Docker Container

**Decision:** Run the Blazor app and MySQL as Docker Compose services on a single Linode VPS. Do not use a managed database service at v1.

**Justification:**
- A managed MySQL service (e.g., Linode Managed Databases) adds ~$30–60/month of cost with no meaningful operational benefit at single-organiser / hobby-to-small-business scale.
- Docker Compose keeps the entire stack reproducible and easy to redeploy or migrate.
- The 5-active-events cap per user is a hard architectural scale ceiling for v1; a single VPS with 4 GB RAM handles this comfortably.

**Tradeoff accepted:** The operator is responsible for MySQL backup. The mitigation is a daily `mysqldump` cron job in a companion container writing to Linode Object Storage. This must be implemented before production launch.

### 2.5 Linode Object Storage (S3-Compatible)

**Decision:** Store only generated PDF files in Object Storage. No application state lives here.

**Justification:**
- PDFs are binary blobs that do not belong in a relational database (LONGBLOB) — they pollute backup size and slow restore time.
- Linode Object Storage is S3-compatible, so the `AWSSDK.S3` NuGet package works without modification. No vendor-specific SDK needed.
- Generated PDFs are accessed by download URL. The `ArchivedEvent` record stores the Object Storage key, and the server generates a presigned URL on demand.

**Tradeoff accepted:** Object Storage adds an external dependency for PDF retrieval. If Object Storage is unavailable, PDF downloads fail, but the app otherwise continues to function. This is an acceptable failure mode.

### 2.6 Gmail SMTP

**Decision:** Use Gmail SMTP with an App Password for transactional email at v1.

**Justification:**
- At projected user volumes (tens to low hundreds of registered organisers), Gmail SMTP's rate limits (500 emails/day for standard Gmail, 2000/day for Workspace) are not a concern.
- Zero additional cost. No third-party email service to configure.

**Tradeoff accepted:** Gmail SMTP is not suitable for high-volume transactional email and has deliverability risks if the account is flagged. At scale, migrate to SendGrid or AWS SES. The migration is a one-line `IEmailSender` swap — the interface hides the provider.

---

## 3. Blazor Render Mode Strategy

This is the most consequential front-end architectural decision because it determines the MAUI Hybrid upgrade path.

### 3.1 The MAUI Constraint

.NET MAUI Hybrid runs a `BlazorWebView` inside a native shell. The `BlazorWebView` executes Blazor components **in-process** using the WebAssembly runtime. This means:
- Components used in MAUI **cannot** use `InteractiveServer` mode — there is no SignalR circuit in a local WebView.
- Components must be compatible with **InteractiveWebAssembly** or **static SSR** rendering.
- All components destined for MAUI must live in a `RazorClassLibrary` (RCL) that has no server-side dependencies injected at runtime.

This is the single principle that governs all render mode decisions below.

### 3.2 Render Mode Assignment

| Page / Component | Render Mode | Rationale |
|---|---|---|
| Landing page | Static SSR | Pure marketing/navigation. No interactivity needed. Fast initial load. SEO-friendly. |
| Login / Register / Password Reset | Static SSR + Enhanced Navigation | Identity UI pages use server-side form posts. No WebSocket overhead needed. |
| Email confirmation | Static SSR | One-shot server callback page. |
| Account Dashboard | Static SSR | List of events with navigation links. Fetched once per page load. No reactive state. |
| Tournament Setup (participant entry, seeding) | **InteractiveServer** | Stateful: participant list grows, bracket reshuffles, preview updates. Rapid UI feedback required. Server-side signalR circuit is acceptable here (web-only in v1). |
| Tournament Bracket / Match Recording | **InteractiveServer** | Rich stateful bracket UI. Result edits must cascade immediately. Server-side state management is simpler than client-side for complex bracket re-computation. |
| League Setup (participant entry) | **InteractiveServer** | Same rationale as Tournament Setup. |
| League Fixture List / Result Entry | **InteractiveServer** | Standings must recalculate on each result entry. InteractiveServer keeps standings logic on server where it belongs. |
| Shared Read-Only View | Static SSR | Viewer loads state, reads it, refreshes manually. No interactivity. Must be publicly accessible without a circuit. |
| PDF Export trigger | Static SSR + HTTP POST | Fire-and-forget server action. No complex client state needed. |
| Admin / Error pages | Static SSR | No interactivity needed. |

### 3.3 MAUI Readiness Preparation (v1 Actions)

Even though MAUI is out of scope for v1, these structural decisions must be made now to avoid a full rewrite later:

1. **Extract all interactive components into a `TourneyGen.Components` RazorClassLibrary.** The Web App project references this RCL. Services are injected via interfaces. The RCL has zero direct EF Core or HttpClient dependencies — it depends only on service abstractions.

2. **Mark InteractiveServer components with `@rendermode InteractiveServer` at the page level, not globally.** Global InteractiveServer on the Router blocks the MAUI migration. Per-component or per-page render mode assignment means the same RCL can be used in MAUI with `InteractiveWebAssembly` substituted for `InteractiveServer`.

3. **Service interfaces must be MAUI-safe.** Services injected into components in the RCL must have `interface` contracts. The Web App wires up EF Core-backed implementations. The MAUI app will wire up HTTP-backed implementations talking to a backend API. This is the Hexagonal/Ports-and-Adapters pattern applied to the UI layer.

4. **Do not use `IJSRuntime` in business components.** JS interop is available in both Blazor Web and MAUI Hybrid, but coupling components to `IJSRuntime` makes them harder to test and slightly more fragile in the MAUI context. Prefer CSS and Blazor-native mechanisms.

### 3.4 Recommended Render Mode — Summary Decision

**Use InteractiveServer for all stateful event management UI. Use Static SSR for everything else.** Do not use InteractiveWebAssembly or InteractiveAuto in v1. The added WASM download size, the need to ship a separate WASM bundle, and the complexity of pre-rendering with client-side hydration are not justified at this scale. When MAUI is introduced, the relevant components are migrated to WASM in the RCL; the web app adds an API layer at that point.

---

## 4. Project Structure

### 4.1 Solution Layout

```
TourneyGen.sln
|
+-- src/
|   +-- TourneyGen.Web/                  # ASP.NET Core + Blazor Web App (the host)
|   |   +-- Components/
|   |   |   +-- Layout/                  # MainLayout, NavMenu
|   |   |   +-- Pages/                   # Route-level pages (thin shells only)
|   |   |   +-- App.razor
|   |   |   +-- Routes.razor
|   |   +-- wwwroot/                     # Static assets
|   |   +-- Program.cs
|   |   +-- appsettings.json
|   |   +-- appsettings.Production.json
|   |
|   +-- TourneyGen.Components/           # RazorClassLibrary — all reusable UI
|   |   +-- Tournament/
|   |   |   +-- TournamentSetup.razor
|   |   |   +-- BracketView.razor
|   |   |   +-- MatchResultEditor.razor
|   |   +-- League/
|   |   |   +-- LeagueSetup.razor
|   |   |   +-- FixtureList.razor
|   |   |   +-- StandingsTable.razor
|   |   +-- Shared/
|   |   |   +-- ParticipantEntry.razor
|   |   |   +-- ConfirmDialog.razor
|   |   |   +-- EventCard.razor
|   |   +-- wwwroot/                     # Component-scoped CSS
|   |
|   +-- TourneyGen.Application/          # Application services (business logic)
|   |   +-- Tournaments/
|   |   |   +-- ITournamentService.cs
|   |   |   +-- TournamentService.cs
|   |   |   +-- BracketEngine.cs
|   |   +-- Leagues/
|   |   |   +-- ILeagueService.cs
|   |   |   +-- LeagueService.cs
|   |   |   +-- StandingsCalculator.cs
|   |   |   +-- FixtureGenerator.cs
|   |   +-- Sharing/
|   |   |   +-- ISharingService.cs
|   |   |   +-- SharingService.cs
|   |   +-- Export/
|   |   |   +-- IExportService.cs
|   |   |   +-- PdfExportService.cs
|   |   +-- Identity/
|   |       +-- IGuestMigrationService.cs
|   |       +-- GuestMigrationService.cs
|   |
|   +-- TourneyGen.Domain/               # Domain entities and value objects
|   |   +-- Entities/
|   |   |   +-- Tournament.cs
|   |   |   +-- League.cs
|   |   |   +-- Participant.cs
|   |   |   +-- TournamentMatch.cs
|   |   |   +-- LeagueFixture.cs
|   |   |   +-- LeagueResult.cs
|   |   |   +-- SharedLink.cs
|   |   |   +-- ArchivedEvent.cs
|   |   +-- Enums/
|   |   |   +-- EventStatus.cs
|   |   |   +-- MatchOutcome.cs
|   |   |   +-- EventType.cs
|   |   |   +-- TiebreakerMethod.cs
|   |   +-- Interfaces/
|   |       +-- (repository contracts if ever needed)
|   |
|   +-- TourneyGen.Infrastructure/       # EF Core, Identity, S3, SMTP
|       +-- Data/
|       |   +-- AppDbContext.cs
|       |   +-- Migrations/
|       |   +-- Configurations/          # IEntityTypeConfiguration<T> per entity
|       +-- Identity/
|       |   +-- ApplicationUser.cs
|       |   +-- EmailSender.cs           # IEmailSender implementation (Gmail SMTP)
|       +-- Storage/
|       |   +-- S3StorageService.cs      # IStorageService implementation
|       +-- DependencyInjection.cs       # Extension method: AddInfrastructure(this IServiceCollection)
|
+-- tests/
|   +-- TourneyGen.Application.Tests/    # Unit tests for services and engines
|   +-- TourneyGen.Domain.Tests/         # Unit tests for domain logic
|   +-- TourneyGen.Integration.Tests/    # EF Core + real DB integration tests
|
+-- docker/
|   +-- docker-compose.yml
|   +-- docker-compose.override.yml      # Dev overrides (local ports, no TLS)
|   +-- nginx/
|       +-- nginx.conf                   # Reverse proxy config
|
+-- docs/
    +-- architecture/
        +-- adr/
        |   +-- README.md
        +-- diagrams/
```

### 4.2 Dependency Direction

The Dependency Rule (*Clean Architecture*, Martin) is enforced:

```
TourneyGen.Web  ---------> TourneyGen.Components
     |                            |
     v                            v
TourneyGen.Infrastructure  --> TourneyGen.Application
                                     |
                                     v
                              TourneyGen.Domain
```

- `Domain` has zero external dependencies — no EF Core, no ASP.NET, no third-party packages.
- `Application` depends on `Domain` and defines service interfaces. It does not reference EF Core or Infrastructure.
- `Infrastructure` implements interfaces defined in `Application` and `Domain`. It owns EF Core, Pomelo, Identity, AWSSDK.S3, and MailKit.
- `Web` composes the DI container and references `Infrastructure` (to register implementations) and `Components` (to render pages).
- `Components` RCL depends on `Application` interfaces only — this is what makes MAUI migration viable.

### 4.3 Namespace Convention

```
TourneyGen.Domain.Entities
TourneyGen.Domain.Enums
TourneyGen.Application.Tournaments
TourneyGen.Application.Leagues
TourneyGen.Application.Sharing
TourneyGen.Application.Export
TourneyGen.Infrastructure.Data
TourneyGen.Infrastructure.Identity
TourneyGen.Infrastructure.Storage
TourneyGen.Web.Components.Pages
TourneyGen.Components.Tournament
TourneyGen.Components.League
TourneyGen.Components.Shared
```

---

## 5. Data Model

### 5.1 Entity Relationship Overview

```
ApplicationUser (ASP.NET Identity)
  |
  +--< Tournament (0..5 active + N archived)
  |       |
  |       +--< Participant
  |       |       (DisplayName, locked after start)
  |       |
  |       +--< TournamentMatch
  |       |       (Round, Position, WinnerId, LoserId)
  |       |
  |       +-- SharedLink (0..1)
  |       +-- ArchivedEvent (0..1, created on completion)
  |
  +--< League (0..5 active combined with tournaments)
          |
          +--< Participant
          |       (DisplayName, locked after start)
          |
          +--< LeagueFixture
          |       (HomeParticipantId, AwayParticipantId, Outcome)
          |
          +-- SharedLink (0..1)
          +-- ArchivedEvent (0..1, created on completion)
```

Note: `Participant` is scoped per event. A participant in Tournament A and a participant with the same name in Tournament B are distinct rows. There is no cross-event participant identity.

### 5.2 Entity Definitions

#### ApplicationUser
Extends `IdentityUser`. Minimal additions:

```
ApplicationUser
  Id               : string (GUID) [PK, inherited from IdentityUser]
  DisplayName      : string (100) [nullable — set on first login or profile setup]
  CreatedAt        : DateTime [UTC]
```

No custom fields beyond what Identity provides. Role-based features are not used in v1 (there is only the Organiser role).

#### Tournament

```
Tournament
  Id               : Guid [PK]
  OrganiserId      : string [FK -> ApplicationUser.Id, NOT NULL]
  Name             : string (200) [NOT NULL]
  Status           : EventStatus [NOT NULL]  -- Draft | Active | Complete
  ParticipantCount : int [NOT NULL]           -- Must be 4/8/16/32/64
  IsLocked         : bool [NOT NULL]          -- true once Started
  CreatedAt        : DateTime [UTC, NOT NULL]
  StartedAt        : DateTime? [UTC, nullable]
  CompletedAt      : DateTime? [UTC, nullable]

  -- Navigation
  Organiser        : ApplicationUser
  Participants     : ICollection<Participant>
  Matches          : ICollection<TournamentMatch>
  SharedLink       : SharedLink?
  ArchivedEvent    : ArchivedEvent?
```

Indexes:
- `(OrganiserId, Status)` — for the dashboard query ("give me this user's active/archived events")
- `(OrganiserId, Status, Id)` — covering index for the active-event count enforcement

#### League

```
League
  Id               : Guid [PK]
  OrganiserId      : string [FK -> ApplicationUser.Id, NOT NULL]
  Name             : string (200) [NOT NULL]
  Status           : EventStatus [NOT NULL]  -- Draft | Active | Complete
  ParticipantCount : int [NOT NULL]           -- 4..64
  IsLocked         : bool [NOT NULL]
  TiebreakerMethod : TiebreakerMethod? [nullable until resolution triggered]
  CreatedAt        : DateTime [UTC, NOT NULL]
  StartedAt        : DateTime?
  CompletedAt      : DateTime?

  -- Navigation
  Organiser        : ApplicationUser
  Participants     : ICollection<Participant>
  Fixtures         : ICollection<LeagueFixture>
  SharedLink       : SharedLink?
  ArchivedEvent    : ArchivedEvent?
```

Indexes:
- `(OrganiserId, Status)` — dashboard query
- Same covering index as Tournament for active-event cap enforcement

#### Participant

```
Participant
  Id               : Guid [PK]
  DisplayName      : string (50) [NOT NULL]
  TournamentId     : Guid? [FK -> Tournament.Id, nullable]
  LeagueId         : Guid? [FK -> League.Id, nullable]
  CreatedAt        : DateTime [UTC, NOT NULL]
```

Constraint: exactly one of `TournamentId` or `LeagueId` must be non-null. This is enforced at the application layer (a check constraint is added in the EF Core configuration).

Indexes:
- `(TournamentId)` — participant lookup per tournament
- `(LeagueId)` — participant lookup per league

#### TournamentMatch

```
TournamentMatch
  Id               : Guid [PK]
  TournamentId     : Guid [FK -> Tournament.Id, NOT NULL]
  Round            : int [NOT NULL]           -- 1 = first round, N = final
  Position         : int [NOT NULL]           -- slot within round (0-indexed)
  Participant1Id   : Guid? [FK -> Participant.Id, nullable]  -- null = BYE
  Participant2Id   : Guid? [FK -> Participant.Id, nullable]  -- null = BYE
  WinnerId         : Guid? [FK -> Participant.Id, nullable]  -- null = not yet played
  LoserId          : Guid? [FK -> Participant.Id, nullable]
  PlayedAt         : DateTime? [UTC]
```

Indexes:
- `(TournamentId, Round)` — load all matches for a round
- `(TournamentId)` — load full bracket

Uniqueness: `(TournamentId, Round, Position)` — unique constraint, no two matches can occupy the same slot.

#### LeagueFixture

```
LeagueFixture
  Id               : Guid [PK]
  LeagueId         : Guid [FK -> League.Id, NOT NULL]
  HomeParticipantId: Guid [FK -> Participant.Id, NOT NULL]
  AwayParticipantId: Guid [FK -> Participant.Id, NOT NULL]
  Leg              : int [NOT NULL]           -- 1 or 2 (double round-robin)
  Outcome          : MatchOutcome? [nullable] -- HomeWin | AwayWin | Draw | null
  PlayedAt         : DateTime? [UTC]
```

Indexes:
- `(LeagueId)` — load all fixtures for standings computation
- `(LeagueId, HomeParticipantId)` — head-to-head tiebreaker query
- `(LeagueId, AwayParticipantId)` — head-to-head tiebreaker query

Uniqueness: `(LeagueId, HomeParticipantId, AwayParticipantId, Leg)` — a given home/away pairing in a given leg is unique.

#### SharedLink

```
SharedLink
  Id               : Guid [PK]
  Token            : string (64) [NOT NULL, UNIQUE]  -- random URL-safe token
  TournamentId     : Guid? [FK -> Tournament.Id, nullable]
  LeagueId         : Guid? [FK -> League.Id, nullable]
  CreatedAt        : DateTime [UTC, NOT NULL]
```

Constraint: exactly one of `TournamentId` or `LeagueId` must be non-null (application-enforced).

Indexes:
- `(Token)` — unique index, primary lookup path for shared view resolution
- `(TournamentId)` — cascade delete
- `(LeagueId)` — cascade delete

#### ArchivedEvent

```
ArchivedEvent
  Id               : Guid [PK]
  OrganiserId      : string [FK -> ApplicationUser.Id, NOT NULL]
  EventType        : EventType [NOT NULL]     -- Tournament | League
  TournamentId     : Guid? [FK -> Tournament.Id, nullable]
  LeagueId         : Guid? [FK -> League.Id, nullable]
  Name             : string (200) [NOT NULL]  -- denormalised for tombstone display
  CompletedAt      : DateTime [UTC, NOT NULL]
  PdfStorageKey    : string (500)? [nullable] -- S3 object key, null until PDF generated
```

The `ArchivedEvent` record is a materialised tombstone created when an event reaches `Status = Complete`. If the event is hard-deleted, the `ArchivedEvent` is cascade-deleted too (its FK has ON DELETE CASCADE). The `PdfStorageKey` stores the S3 object path so the app can generate a presigned download URL without querying S3 directory listings.

Indexes:
- `(OrganiserId, CompletedAt DESC)` — dashboard archived list, most recent first

### 5.3 Enumerations

```csharp
enum EventStatus   { Draft, Active, Complete }
enum MatchOutcome  { HomeWin, AwayWin, Draw }         // League only
enum EventType     { Tournament, League }
enum TiebreakerMethod { AutoHeadToHead, GenerateBracket, ManualSelection, CoinFlip }
```

### 5.4 Active Event Cap Enforcement

The 5-event limit is enforced at the application layer in `TournamentService.CreateAsync` and `LeagueService.CreateAsync`:

```
SELECT COUNT(*) FROM Tournaments WHERE OrganiserId = @id AND Status != 'Complete'
UNION ALL
SELECT COUNT(*) FROM Leagues WHERE OrganiserId = @id AND Status != 'Complete'
```

This is a simple count query on indexed columns. It runs within a database transaction with the insert to prevent race conditions (last-writer-wins without the transaction would allow 6 simultaneous creates). The check is **not** done in a Blazor component — it belongs in the service layer.

---

## 6. API / Service Layer

### 6.1 Decision: Direct Service Injection (No Separate Web API)

**Decision:** Blazor components inject application service interfaces directly. There is no separate ASP.NET Core Web API (no controllers, no `/api/` routes) in v1.

**Rationale:** Direct injection is the canonical Blazor pattern for server-side rendering. An internal HTTP API would add serialisation overhead, an extra indirection layer, and authentication token plumbing — all cost with no benefit when the components run on the same server process as the services. See *Patterns of Enterprise Application Architecture* (Fowler) on the "Remote Facade" pattern: introduce remoting only when processes are genuinely separated.

**Tradeoff accepted:** When the MAUI client is built, it will need a Web API (because the MAUI app is a separate process). At that point, the Application service layer is wrapped in a thin API controller layer. The existing service interfaces require no change — the controller becomes the new adapter. This is the Hexagonal Architecture payoff: the port (service interface) is unchanged; only the adapter (controller vs. direct injection) differs.

### 6.2 Service Contracts

**ITournamentService**
- `CreateAsync(organiserId, name, participantCount) -> Tournament`
- `AddParticipantAsync(tournamentId, displayName) -> Participant`
- `ShuffleSeeding(tournamentId) -> IReadOnlyList<Participant>`  (in-memory, persisted on confirm)
- `StartAsync(tournamentId) -> Tournament`  (locks participant list, persists seeding)
- `RecordResultAsync(matchId, winnerId) -> TournamentMatch`
- `EditResultAsync(matchId, newWinnerId, confirmCascade: bool) -> TournamentMatch`
- `RenameParticipantAsync(participantId, newName) -> Participant`
- `DeleteAsync(tournamentId) -> void`
- `GetBracketAsync(tournamentId) -> BracketViewModel`
- `GetByIdAsync(tournamentId) -> Tournament`

**ILeagueService**
- `CreateAsync(organiserId, name, participantCount) -> League`
- `AddParticipantAsync(leagueId, displayName) -> Participant`
- `StartAsync(leagueId) -> League`  (generates all fixtures via FixtureGenerator)
- `RecordFixtureResultAsync(fixtureId, outcome) -> LeagueFixture`
- `EditFixtureResultAsync(fixtureId, newOutcome) -> LeagueFixture`
- `RenameParticipantAsync(participantId, newName) -> Participant`
- `GetStandingsAsync(leagueId) -> IReadOnlyList<StandingRow>`
- `ResolveTiebreakerAsync(leagueId, method, manualWinnerId?) -> League`
- `DeleteAsync(leagueId) -> void`

**ISharingService**
- `GenerateLinkAsync(tournamentId?, leagueId?) -> SharedLink`
- `ResolveTokenAsync(token) -> SharedEventViewModel?`  (null = not found)

**IExportService**
- `GeneratePdfAsync(archivedEventId) -> string`  (returns storage key)
- `GetDownloadUrlAsync(storageKey) -> string`  (presigned S3 URL, 1-hour TTL)

**IGuestMigrationService**
- `MigrateToAccountAsync(guestState: GuestTournamentState, organiserId: string) -> Tournament`

### 6.3 ViewModel Pattern

Service methods return domain entities or purpose-built ViewModels. Components never query `AppDbContext` directly.

`BracketViewModel` and `StandingRow` are lightweight records in `TourneyGen.Application` that carry only what the component needs. This keeps the component layer thin and testable.

---

## 7. Authentication Architecture

### 7.1 Flow Overview

```
+------------------+     +------------------+     +------------------+
|  Email/Password  |     |  Google OAuth    |     |  Guest Mode      |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         v                        v                        v
+--------+------------------------+----+         +---------+----------+
|         ASP.NET Core Identity        |         |  Browser Session   |
|         (Identity Cookie Auth)        |         |  (no server state) |
+--------+---------------------+-------+         +--------------------+
         |                     |
         v                     v
  Email Confirmation      Password Reset
  (Gmail SMTP)            (Gmail SMTP)
         |
         v
  Account Activated
```

### 7.2 Identity Configuration

```csharp
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.SignIn.RequireConfirmedEmail = true;
    options.Password.RequiredLength = 8;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```

### 7.3 Google OAuth Configuration

Google OAuth is registered as an additional external login provider. On first Google sign-in, Identity automatically creates a linked `ApplicationUser` record. Email confirmation is bypassed for OAuth sign-ins (the email is verified by Google).

```csharp
builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        options.ClientId = configuration["Authentication:Google:ClientId"];
        options.ClientSecret = configuration["Authentication:Google:ClientSecret"];
    });
```

Google credentials are stored in environment variables / `appsettings.Production.json` (not in source control). The `.gitignore` must exclude `appsettings.Production.json`.

### 7.4 Session and Cookie Configuration

The Identity cookie is the authentication mechanism. There is no JWT in v1.

```csharp
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Lax;
    options.ExpireTimeSpan = TimeSpan.FromDays(14);
    options.SlidingExpiration = true;
    options.LoginPath = "/account/login";
    options.AccessDeniedPath = "/account/access-denied";
});
```

`SameSite = Lax` (not `Strict`) is required for the Google OAuth redirect to carry the cookie back correctly.

### 7.5 Authorization

Route-level authorization is enforced with the `[Authorize]` attribute on Blazor page components and via `AuthorizeRouteView` in `Routes.razor`. The shared-link read-only view is intentionally unauthenticated (`[AllowAnonymous]`).

The 5-event cap is enforced in the service layer, not in route authorization — it is a business rule, not an access control rule.

---

## 8. Guest Session Strategy

### 8.1 Decision: Browser-Only State via Blazor Component Memory (No Server Storage)

**Decision:** Guest tournament state lives exclusively in the Blazor InteractiveServer component's in-memory state on the server-side circuit. It is never written to the database. No `HttpContext.Session`, no `ProtectedSessionStorage`, no temporary DB records.

**Rationale:**
- The PRD explicitly states: *"Session data is held in browser memory; data is lost permanently when the tab is closed."* The most honest implementation of this constraint is a Blazor circuit with in-memory state. When the tab closes, the SignalR circuit disconnects, the component is disposed, and the state is gone. No cleanup needed.
- A temporary DB record approach would require a garbage-collection job to remove abandoned guest sessions, adding operational complexity with no user-visible benefit.
- `ProtectedSessionStorage` (browser session storage) cannot hold the full tournament state (bracket tree with match relationships) ergonomically; it stores serialised strings and introduces async serialisation on every state change.

**Tradeoff accepted:** If the server restarts (e.g., deployment), the guest's in-progress tournament is lost. This is consistent with the "tab closed = data lost" contract. A deployment notification UI (e.g., "reconnecting...") is sufficient mitigation.

### 8.2 GuestTournamentState

A C# record held in the InteractiveServer component's field:

```csharp
public record GuestTournamentState(
    List<string> ParticipantNames,
    int[][] BracketSeeding,          // index into ParticipantNames
    List<GuestMatchResult> Results
);
```

This state object is passed to `IGuestMigrationService.MigrateToAccountAsync` on login.

### 8.3 Migration Flow

1. Guest completes or partially completes a tournament in session.
2. Guest clicks "Log In" or "Create Account."
3. After successful authentication, the middleware detects `GuestTournamentState` in the current circuit scope.
4. `GuestMigrationService` receives the state and the authenticated user's ID.
5. The service creates a full `Tournament` + `Participant` + `TournamentMatch` record set in the database, with `Status = Active` (or `Complete` if the guest finished it).
6. The user is redirected to the migrated tournament page.

The migration must occur in the same request/circuit as the sign-in callback. The `GuestTournamentState` is passed to the migration service via a scoped service registered before the auth redirect happens.

### 8.4 Implementation Note on Circuit Lifetime

The `GuestTournamentState` lives in the component, which is scoped to the SignalR circuit. The migration service is called from a Blazor event handler before navigation. This avoids any cross-circuit state transfer problem.

---

## 9. Sharing Architecture

### 9.1 Token Generation

A shared link token is a 32-byte cryptographically random value encoded as a URL-safe Base64 string (43 characters). Generated using `RandomNumberGenerator.GetBytes(32)`.

```csharp
var bytes = RandomNumberGenerator.GetBytes(32);
var token = Base64UrlTextEncoder.Encode(bytes); // Microsoft.AspNetCore.WebUtilities
```

The token is stored in the `SharedLink.Token` column which has a unique index. The link URL pattern is:

```
https://tourneygen.example.com/share/{token}
```

### 9.2 Resolution

The `/share/{token}` route is a static SSR Blazor page with `[AllowAnonymous]`. On render, it calls `ISharingService.ResolveTokenAsync(token)`.

- If `token` resolves to an existing event: renders the read-only view.
- If `token` does not exist (deleted event or invalid token): renders a generic "not found" page with no information about what was expected.

No 301/302 redirect — the read-only view renders at the `/share/{token}` URL itself. This is simpler and avoids an extra round-trip.

### 9.3 "Last Updated" Timestamp

The read-only view displays a "Last updated" timestamp. This comes from the latest `PlayedAt` value across all matches/fixtures in the event, or `StartedAt` if no results have been recorded yet. This is computed in `SharingService.ResolveTokenAsync` — it is not a separate stored field.

### 9.4 Security Considerations

- The token space is 2^256 — brute-force enumeration is computationally infeasible.
- The shared view is intentionally public (no auth). The "security" is the unguessable token, not authentication. This matches the stated PRD behaviour.
- The `/share/` route does not reveal whether a token ever existed (uniform "not found" response for both invalid and deleted-event tokens). This prevents enumeration timing attacks.
- No sensitive organiser information (email, account details) is ever displayed in the shared view.

### 9.5 Deletion Cascade

On hard delete of a Tournament or League, the `SharedLink` row is deleted via EF Core cascade delete (configured in `OnDelete(DeleteBehavior.Cascade)`). After deletion, the token simply does not exist in the `SharedLink` table, so `/share/{token}` naturally returns "not found."

---

## 10. PDF Export

### 10.1 Library Decision: QuestPDF

**Decision:** Use QuestPDF (MIT license, free for non-commercial and small commercial use; the community licence covers an application at this scale).

**Rationale:**
- QuestPDF has a fluent C# API with no HTML-to-PDF pipeline, no headless Chrome dependency, no Puppeteer. It generates PDFs directly from a document model.
- It runs fully server-side in the ASP.NET Core process — no sidecar process, no external service call.
- Wkhtmltopdf (a common alternative) requires a native binary on the server and has known rendering inconsistencies on ARM/x64 Docker containers. PdfSharp requires more low-level layout code. QuestPDF is the cleanest fit.
- iTextSharp's AGPL license would require the application itself to be open-source; QuestPDF avoids this legal constraint.

**Tradeoff accepted:** QuestPDF's community licence requires attribution in the PDF footer for non-commercial use. This is acceptable. If the application becomes commercial, the indie licence ($99/year) removes this requirement.

### 10.2 PDF Generation Flow

1. Organiser clicks "Export PDF" on an archived event page.
2. Blazor component calls `IExportService.GeneratePdfAsync(archivedEventId)`.
3. `PdfExportService` queries the event data (participants, results, final standings/bracket outcome).
4. QuestPDF generates the PDF document in memory as a `byte[]`.
5. `PdfExportService` uploads the `byte[]` to Linode Object Storage using `AWSSDK.S3`. The S3 key is `pdfs/{organiserId}/{archivedEventId}.pdf`.
6. The S3 key is persisted to `ArchivedEvent.PdfStorageKey`.
7. The service returns a presigned URL (1-hour TTL) for immediate download.

On subsequent "Download PDF" requests (where `PdfStorageKey` is already set), the service skips generation and goes directly to presigned URL generation.

### 10.3 PDF Content

The PDF contains:
- Event name, type, and completion date.
- Winner(s) prominently displayed at the top.
- Full results: bracket match history (tournament) or final standings table (league).
- "Generated by TourneyGen" footer (community licence attribution).

PDF layout is kept simple — no images, no custom fonts beyond the QuestPDF system font defaults. This avoids font-embedding complexity on the Linux Docker container.

### 10.4 Object Storage Configuration

The S3 bucket is configured as **private** (no public-read ACL). PDFs are accessed only via presigned URLs. This means a PDF URL expires and cannot be hotlinked permanently, which is intentional — the organiser must be logged in to regenerate a fresh URL.

```csharp
var request = new GetPreSignedUrlRequest
{
    BucketName = _options.BucketName,
    Key = storageKey,
    Expires = DateTime.UtcNow.AddHours(1),
    Verb = HttpVerb.GET
};
return _s3Client.GetPreSignedURL(request);
```

---

## 11. Key Non-Functional Requirements

### 11.1 Scalability

TourneyGen v1 is deliberately not designed to scale horizontally. The constraints that make vertical scaling sufficient:

- 5 active events per user hard cap limits per-user database load.
- No realtime push; the shared view is pull-only (manual refresh).
- InteractiveServer circuits are created only during event management; read-only pages (dashboard, shared view) are stateless SSR.
- A single Linode VPS (4 vCPU, 8 GB RAM) can handle hundreds of concurrent organisers within these constraints. A SignalR circuit uses approximately 250 KB of server memory at baseline; 1,000 concurrent organisers ≈ 250 MB, well within budget.

If the application grows beyond ~5,000 registered users, the scaling strategy is: sticky-session load balancer + Redis-backed SignalR backplane + read replica for the dashboard/shared-view queries. These are additive changes to the existing architecture, not a redesign.

### 11.2 Security

**CSRF:** Blazor's SSR form handling uses the built-in `AntiforgeryToken` mechanism. The `InteractiveServer` components do not use HTML form posts; they use Blazor event callbacks, which are protected by the SignalR circuit's session-bound nature (a circuit is tied to its authenticated user).

**Input Sanitization:** Participant names (50-char max) are stored as plain text and HTML-encoded on output by Blazor's renderer by default (`@name` binding is always HTML-encoded). No HTML is accepted from user input anywhere in the system. EF Core parameterises all queries, eliminating SQL injection.

**Rate Limiting:** Apply ASP.NET Core's built-in rate limiting middleware (`RateLimiterMiddleware`) on the `/account/login` and `/account/register` routes. A fixed-window policy of 10 requests per minute per IP is sufficient to deter credential stuffing at this scale.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("auth", policy =>
    {
        policy.PermitLimit = 10;
        policy.Window = TimeSpan.FromMinutes(1);
        policy.QueueLimit = 0;
    });
});
```

**HTTPS:** Enforced at the Nginx reverse proxy layer. The ASP.NET Core app binds to HTTP internally (port 8080) and trusts the `X-Forwarded-Proto` header from Nginx. `app.UseHsts()` and `app.UseHttpsRedirection()` are **not** called inside the app (Nginx handles TLS termination).

**Secrets Management:** Database connection strings, Google OAuth credentials, Gmail App Password, and S3 keys are supplied via environment variables in `docker-compose.yml` (via a `.env` file excluded from source control). They are never committed to the repository.

**Account Lockout:** Enabled by default in Identity configuration (5 failed attempts = 15-minute lockout).

### 11.3 Responsiveness (Web-First, MAUI-Ready)

The application uses Bootstrap 5 (included in the default Blazor Web App template) for responsive layout. All pages are mobile-responsive by design.

The bracket view (visually the most complex component) must render acceptably on screens as narrow as 375px. For large brackets (64 participants, 6 rounds), a horizontal scroll container is used rather than attempting to compress the bracket — forced compression makes match names illegible. This is a UX decision, not a technical constraint.

### 11.4 Observability

Logging uses the built-in `ILogger<T>` with structured logging. For v1, logs are written to `stdout` (captured by Docker) and to a rolling file via Serilog's `WriteTo.File` sink. Log levels:
- `Information` for all service operation completions.
- `Warning` for business rule violations (e.g., attempted 6th event creation, invalid token resolution).
- `Error` for exceptions.

No distributed tracing or metrics platform is required at v1. Add OpenTelemetry when a second service or meaningful traffic exists.

---

## 12. Future MAUI Hybrid Path

### 12.1 What Changes

When MAUI Hybrid is introduced, the additions are:

1. A new `TourneyGen.Maui` project: a .NET MAUI application with a `BlazorWebView` and a native shell (tab bar, platform lifecycle hooks).
2. A new `TourneyGen.Api` project: an ASP.NET Core Minimal API (or Web API) layer that exposes the Application service layer over HTTP/JSON. This is the adapter that MAUI needs.
3. HTTP-backed implementations of the service interfaces (e.g., `HttpTournamentService : ITournamentService`) in a new `TourneyGen.Client` project, injected in the MAUI app.

### 12.2 What Does Not Change

- `TourneyGen.Domain` — zero changes.
- `TourneyGen.Application` — zero changes (service interfaces are the stable port).
- `TourneyGen.Components` — the RCL components are reused as-is in `BlazorWebView`. Render mode annotations are removed (MAUI uses `InteractiveAuto` by default for the `BlazorWebView`). If per-component `@rendermode` directives are used rather than global configuration, they are simply not applied in the MAUI context.
- `TourneyGen.Infrastructure` — zero changes (EF Core, S3, SMTP all remain on the server).

### 12.3 Migration Path Timeline

| Phase | Action |
|---|---|
| v1 (Now) | Build with direct service injection, InteractiveServer. Extract all UI into `TourneyGen.Components` RCL. |
| v2 Pre-MAUI | Add `TourneyGen.Api` (Minimal API wrapping Application services). Validate service interfaces are fully expressible over HTTP. |
| v2 MAUI | Add `TourneyGen.Maui` + `TourneyGen.Client`. Wire `BlazorWebView` to `TourneyGen.Components`. |

The key insight from *Clean Architecture* (Martin) applies here: the use cases (Application layer) and entities (Domain layer) are the stable core. The delivery mechanism (Blazor Web vs. MAUI Hybrid) is a plug-in. The current architecture preserves this separation by design.

---

## 13. Deployment Architecture

### 13.1 Docker Compose Layout

```yaml
# docker/docker-compose.yml

version: "3.9"

services:

  app:
    build:
      context: ..
      dockerfile: src/TourneyGen.Web/Dockerfile
    image: tourneygen-web:latest
    restart: unless-stopped
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__DefaultConnection=${DB_CONNECTION_STRING}
      - Authentication__Google__ClientId=${GOOGLE_CLIENT_ID}
      - Authentication__Google__ClientSecret=${GOOGLE_CLIENT_SECRET}
      - Email__SmtpHost=smtp.gmail.com
      - Email__SmtpPort=587
      - Email__Username=${GMAIL_USERNAME}
      - Email__Password=${GMAIL_APP_PASSWORD}
      - Storage__BucketName=${S3_BUCKET_NAME}
      - Storage__AccessKey=${S3_ACCESS_KEY}
      - Storage__SecretKey=${S3_SECRET_KEY}
      - Storage__ServiceUrl=https://us-east-1.linodeobjects.com
    depends_on:
      db:
        condition: service_healthy
    networks:
      - internal

  db:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      - MYSQL_DATABASE=tourneygen
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/www/certbot:/var/www/certbot:ro
    depends_on:
      - app
    networks:
      - internal
      - external

  backup:
    image: mysql:8.0
    restart: "no"
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
    volumes:
      - ./backup.sh:/backup.sh:ro
      - backup_tmp:/backup
    entrypoint: /backup.sh
    depends_on:
      - db
    networks:
      - internal

volumes:
  mysql_data:
  backup_tmp:

networks:
  internal:
    driver: bridge
  external:
    driver: bridge
```

### 13.2 Nginx Configuration

Nginx handles TLS termination (Let's Encrypt certificates via Certbot), HTTP-to-HTTPS redirect, and proxying to the app container. The `X-Forwarded-Proto` and `X-Forwarded-For` headers are passed so the ASP.NET Core app can reconstruct correct absolute URLs for OAuth callbacks and email confirmation links.

```nginx
server {
    listen 80;
    server_name tourneygen.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name tourneygen.example.com;

    ssl_certificate /etc/letsencrypt/live/tourneygen.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tourneygen.example.com/privkey.pem;

    location / {
        proxy_pass         http://app:8080;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";   # Required for SignalR WebSocket
        proxy_set_header   Host $host;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;                  # Long timeout for SignalR circuits
    }
}
```

The `proxy_http_version 1.1` + `Upgrade` + `Connection upgrade` headers are mandatory for Blazor's SignalR WebSocket transport. Without them, Blazor Interactive falls back to Long Polling, which is significantly less efficient.

### 13.3 Database Migrations at Startup

EF Core migrations run automatically at application startup in the development environment:

```csharp
// In Program.cs, development only:
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
```

In production, migrations are applied as a pre-deployment step (a separate `docker run` command with `dotnet ef database update` or a migration runner container), not at application startup. Running migrations at startup in production introduces startup latency and risks applying a partially completed migration under concurrent traffic.

### 13.4 Backup Strategy

The `backup` service runs a daily `mysqldump` via cron on the VPS host (outside Docker Compose) that pipes output to an S3 upload. The backup bucket is separate from the PDF bucket and has a 30-day lifecycle policy.

```bash
# /etc/cron.daily/tourneygen-backup
mysqldump -h 127.0.0.1 -P 3306 -u root -p${DB_ROOT_PASSWORD} tourneygen \
  | gzip \
  | aws s3 cp - s3://${BACKUP_BUCKET}/$(date +%Y-%m-%d).sql.gz \
      --endpoint-url https://us-east-1.linodeobjects.com
```

This is a minimum viable backup. Point-in-time recovery is not available with this approach. If data criticality increases, add MySQL binary log shipping to a second location.

### 13.5 Application Dockerfile

```dockerfile
# src/TourneyGen.Web/Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.sln .
COPY src/ src/
RUN dotnet restore
RUN dotnet publish src/TourneyGen.Web/TourneyGen.Web.csproj \
    -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "TourneyGen.Web.dll"]
```

The multi-stage build keeps the final image lean (the SDK layer is discarded). The runtime image is `mcr.microsoft.com/dotnet/aspnet:8.0` (not the SDK image), which is approximately 220 MB.

---

## Appendix: Key Architectural Decisions Summary

| Decision | Choice | Primary Tradeoff |
|---|---|---|
| Architecture style | Modular monolith | Operational simplicity over independent deployability |
| Blazor render mode | InteractiveServer for state, SSR for rest | Server memory per circuit vs. WASM download size |
| Service layer approach | Direct injection, no Web API (v1) | MAUI will require API layer addition in v2 |
| Guest state storage | Circuit in-memory (no DB) | Lost on server restart; no cleanup job needed |
| PDF library | QuestPDF | Community licence attribution required |
| PDF storage | Object Storage, private + presigned URL | URLs expire; not permanent hotlinks |
| Database | MySQL in Docker, single VPS | No managed HA; operator manages backups |
| Auth | ASP.NET Core Identity + Google OAuth | Full Identity schema overhead vs. hand-rolled simplicity |
| Shared link security | Unguessable random token, no auth | Anyone with the link can view (intentional) |
| MAUI readiness | RCL for all UI, interface abstractions for services | Extra project in solution; minor DI complexity |
