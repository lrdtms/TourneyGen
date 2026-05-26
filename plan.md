# TourneyGen v1 — Implementation Plan

---

## Phase 1 — Solution Scaffold

### Step 1.1 — Create .NET 8 Solution and Projects

Create the solution file and all five C# projects in one pass.

```
dotnet new sln -n TourneyGen
dotnet new classlib -n TourneyGen.Domain -o src/TourneyGen.Domain
dotnet new classlib -n TourneyGen.Application -o src/TourneyGen.Application
dotnet new classlib -n TourneyGen.Infrastructure -o src/TourneyGen.Infrastructure
dotnet new razorclasslib -n TourneyGen.Components -o src/TourneyGen.Components
dotnet new blazor -n TourneyGen.Web -o src/TourneyGen.Web \
    --interactivity Server \
    --all-interactive false \
    --no-https false
dotnet sln add src/TourneyGen.Domain src/TourneyGen.Application \
             src/TourneyGen.Infrastructure src/TourneyGen.Components src/TourneyGen.Web
```

Key decisions:
- Use `blazor` template (the .NET 8/9 Blazor Web App template — the CLI name is `blazor`, not `blazorweb`). This is the unified template supporting mixed render modes.
- `--interactivity Server` — registers only InteractiveServer render mode. Do NOT use `--interactivity Auto` or `--interactivity WebAssembly`; those templates add WASM projects and are not part of this architecture.
- `--all-interactive false` — critical flag. This ensures render mode is applied **per-component/per-page**, not globally. A global InteractiveServer on `Routes.razor` would block the future MAUI migration (architecture section 3.2 explicitly requires per-component render mode assignment). Without this flag the template wires `@rendermode InteractiveServer` onto `Routes.razor`, making every page interactive by default.
- `TourneyGen.Components` uses `razorclasslib` so it can later be consumed by a MAUI shell without modification.
- The `classlib` template for Domain/Application/Infrastructure keeps those projects free of ASP.NET dependencies by default.

> [Blazor Review] The original plan used `dotnet new blazorweb` which is not a valid .NET 8/9 template alias. The correct command is `dotnet new blazor`. The `--interactivity` and `--all-interactive` flags are mandatory for this architecture and were absent. Without `--all-interactive false`, the template applies `@rendermode InteractiveServer` globally on `Routes.razor`, which contradicts the per-page render mode strategy in architecture section 3.2 and breaks MAUI compatibility.

Gotcha: The default `blazor` template creates sample pages (`Counter.razor`, `Weather.razor`). Delete them and the associated `WeatherForecastService` immediately so the scaffold is clean.

### Step 1.2 — Configure Project References

Wire up the dependency graph exactly as defined in the architecture (dependency rule: inner layers never reference outer layers):

```
TourneyGen.Domain            → (no project references)
TourneyGen.Application       → TourneyGen.Domain
TourneyGen.Infrastructure    → TourneyGen.Application, TourneyGen.Domain
TourneyGen.Components        → TourneyGen.Application
TourneyGen.Web               → TourneyGen.Infrastructure, TourneyGen.Components, TourneyGen.Domain
```

Execute with `dotnet add reference` commands. Verify the graph has no cycles (`dotnet build` will catch cycles).

Key decision: `TourneyGen.Components` does NOT reference `TourneyGen.Infrastructure` — this is the MAUI readiness invariant. Components only know about service interfaces.

> [Blazor Review] **RCL wiring into the Blazor Web App host — `_Imports.razor` and namespace configuration.** After adding the project reference from `TourneyGen.Web` to `TourneyGen.Components`, two additional wiring steps are required or the RCL components will not resolve in the Web project's pages:
>
> 1. **Add `@using` directives to `src/TourneyGen.Web/Components/_Imports.razor`:**
>    ```razor
>    @using TourneyGen.Components.Tournament
>    @using TourneyGen.Components.League
>    @using TourneyGen.Components.Shared
>    ```
>    Without these, Razor compilation cannot resolve `<TournamentSetup>`, `<BracketView>`, `<LeagueSetup>`, etc. in the Web host's page components.
>
> 2. **Create `src/TourneyGen.Components/_Imports.razor`** in the RCL itself with the RCL's own `@using` directives:
>    ```razor
>    @using Microsoft.AspNetCore.Components
>    @using Microsoft.AspNetCore.Components.Forms
>    @using Microsoft.AspNetCore.Components.Routing
>    @using Microsoft.AspNetCore.Components.Web
>    @using Microsoft.AspNetCore.Authorization
>    @using TourneyGen.Application.Tournaments
>    @using TourneyGen.Application.Leagues
>    @using TourneyGen.Application.Sharing
>    @using TourneyGen.Application.Identity
>    ```
>    The RCL `_Imports.razor` applies to all `.razor` files within `TourneyGen.Components` only. It does not automatically propagate to the host project — that is why step 1 above (the Web project's `_Imports.razor`) is also required.
>
> 3. **Static assets from the RCL:** If `TourneyGen.Components` includes component-scoped CSS or JS files under its `wwwroot/`, the Web host automatically serves them at the path `/_content/TourneyGen.Components/`. No additional configuration is needed for asset serving. Reference component-scoped CSS files in `App.razor` as:
>    ```html
>    <link rel="stylesheet" href="_content/TourneyGen.Components/TourneyGen.Components.styles.css" />
>    ```
>    The `.styles.css` bundle is generated automatically by the Blazor build tooling when CSS isolation (`.razor.css` files) is used in the RCL.

### Step 1.3 — Add NuGet Packages

**TourneyGen.Domain** — no packages (pure C#).

**TourneyGen.Application:**
- `QuestPDF` (for `IExportService` contract types, if any PDF data objects live here; alternatively keep QuestPDF only in Infrastructure)

**TourneyGen.Infrastructure:**
- `Pomelo.EntityFrameworkCore.MySql` (8.x — matches EF Core 8)
- `Microsoft.EntityFrameworkCore.Design` (for migrations CLI)
- `Microsoft.AspNetCore.Identity.EntityFrameworkCore`
- `AWSSDK.S3`
- `MailKit` (Gmail SMTP via `IEmailSender`)
- `QuestPDF`
- `Serilog.AspNetCore`, `Serilog.Sinks.File`

**TourneyGen.Web:**
- `Microsoft.AspNetCore.Authentication.Google`
- `Microsoft.AspNetCore.RateLimiting` (built-in, no NuGet needed in .NET 8)

**Test projects (add after Phase 1):**
- `xunit`, `Moq`, `FluentAssertions`, `Microsoft.EntityFrameworkCore.InMemory`

Gotcha: Pomelo must be version-matched to the installed MySQL server version (use `ServerVersion.AutoDetect` in dev, pin to `8.0` in prod). Do not use Oracle's `MySql.EntityFrameworkCore` — it lags behind.

> [ASP.NET Review] **Pinned package versions for `TourneyGen.Infrastructure`.** Use these exact versions to ensure EF Core 8 / Pomelo compatibility:
> ```
> Pomelo.EntityFrameworkCore.MySql                    8.0.2
> Microsoft.EntityFrameworkCore.Design                8.0.x  (match EF Core runtime)
> Microsoft.EntityFrameworkCore.Tools                 8.0.x  (add to TourneyGen.Web too, PrivateAssets="all")
> Microsoft.AspNetCore.Identity.EntityFrameworkCore   8.0.x
> AWSSDK.S3                                           3.7.x
> MailKit                                             4.x
> QuestPDF                                            2024.x (latest stable)
> Serilog.AspNetCore                                  8.x
> Serilog.Sinks.File                                  5.x
> ```
> Do NOT mix Pomelo 9.x with EF Core 8 — the Pomelo major version must match the EF Core major version. Also add `Microsoft.EntityFrameworkCore.Tools` (8.0.x) to `TourneyGen.Web` as `<PackageReference ... PrivateAssets="all" />` — without it, `dotnet ef migrations add` fails with "No DbContext was found" when targeting TourneyGen.Web as the startup project.

### Step 1.4 — Create Test Projects

```
dotnet new xunit -n TourneyGen.Domain.Tests -o tests/TourneyGen.Domain.Tests
dotnet new xunit -n TourneyGen.Application.Tests -o tests/TourneyGen.Application.Tests
dotnet new xunit -n TourneyGen.Integration.Tests -o tests/TourneyGen.Integration.Tests
```

Add project references from each test project to the project under test. Add all three to the solution.

### Step 1.5 — Create Directory Skeleton

Create the folder structure defined in architecture section 4.1. All folders are empty placeholders at this stage (use `.gitkeep` files):

```
src/TourneyGen.Domain/Entities/
src/TourneyGen.Domain/Enums/
src/TourneyGen.Domain/Interfaces/
src/TourneyGen.Application/Tournaments/
src/TourneyGen.Application/Leagues/
src/TourneyGen.Application/Sharing/
src/TourneyGen.Application/Export/
src/TourneyGen.Application/Identity/
src/TourneyGen.Infrastructure/Data/Configurations/
src/TourneyGen.Infrastructure/Data/Migrations/
src/TourneyGen.Infrastructure/Identity/
src/TourneyGen.Infrastructure/Storage/
src/TourneyGen.Components/Tournament/
src/TourneyGen.Components/League/
src/TourneyGen.Components/Shared/
src/TourneyGen.Components/wwwroot/
src/TourneyGen.Web/Components/Layout/
src/TourneyGen.Web/Components/Pages/
src/TourneyGen.Web/wwwroot/
docker/nginx/
docs/architecture/adr/
docs/architecture/diagrams/
```

### Step 1.6 — Docker Compose (Development)

Create `docker/docker-compose.yml` using the exact configuration from architecture section 13.1. Create `docker/docker-compose.override.yml` for development with:
- Exposed MySQL port (3306) to localhost for EF Core migrations from the host machine
- No TLS (HTTP only)
- `ASPNETCORE_ENVIRONMENT=Development`

Create `docker/nginx/nginx.conf` from architecture section 13.2. Replace `tourneygen.example.com` with a placeholder and note it must be updated with the real domain.

Create `.env.example` at the repo root listing all required environment variables (never fill in real values). Copy to `.env` locally and add `.env` to `.gitignore`.

Verify `.gitignore` excludes: `*.env`, `appsettings.Production.json`, `appsettings.*.json` (except `appsettings.json` and `appsettings.Development.json`).

> [ASP.NET Review] **Docker Compose healthcheck `start_period` is missing.** The `mysqladmin ping` healthcheck in architecture section 13.1 should include `start_period: 30s`. Without it the healthcheck starts counting failures immediately, MySQL's first-run data-directory initialisation (15–25 seconds) exhausts all retries, and the `app` container's `depends_on: condition: service_healthy` never becomes satisfied. The correct healthcheck block:
> ```yaml
> healthcheck:
>   test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "--user=root", "--password=${DB_ROOT_PASSWORD}"]
>   interval: 10s
>   timeout: 5s
>   retries: 5
>   start_period: 30s
> ```

> [ASP.NET Review] **Add `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true` to the `app` service in `docker-compose.yml`.** ASP.NET Core 8 recognises this environment variable and automatically configures `ForwardedHeadersOptions` to trust `X-Forwarded-For` and `X-Forwarded-Proto` from the Docker internal network. Without this, the OAuth middleware constructs redirect URIs as `http://` (the internal port 8080 scheme) instead of `https://`, causing a `redirect_uri_mismatch` error with Google OAuth and breaking email confirmation links in production.

> [ASP.NET Review] **Connection string format in `.env.example`.** Use the Pomelo/MySqlConnector format:
> ```
> DB_CONNECTION_STRING=Server=db;Port=3306;Database=tourneygen;User=tourneygen_app;Password=CHANGEME;AllowPublicKeyRetrieval=true;SslMode=None;
> ```
> `Server=db` matches the Docker Compose service name. `AllowPublicKeyRetrieval=true` and `SslMode=None` are required for intra-container connections (no TLS on the Docker bridge network).

**MinIO local dev storage:** Also add `minio` and `minio-init` services to `docker-compose.override.yml` so the app has a real S3-compatible bucket locally, mirroring Linode Object Storage in production. No application code changes are needed — only the `Storage__*` environment variables differ per environment.

```yaml
  minio:
    image: minio/minio:latest
    restart: unless-stopped
    command: server /data
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER:-minioadmin}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD:-minioadmin}
    volumes:
      - minio_data:/data
    networks:
      - internal

  minio-init:
    image: minio/mc:latest
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
        until /usr/bin/mc alias set local http://minio:9000 $${MINIO_ROOT_USER:-minioadmin} $${MINIO_ROOT_PASSWORD:-minioadmin}; do
          echo 'Waiting for MinIO...'; sleep 2;
        done;
        /usr/bin/mc mb --ignore-existing local/tourneygen-dev;
        exit 0;
      "
    networks:
      - internal
```

Add `minio_data:` to the top-level `volumes:` key in the override file. Override the `app` service storage env vars in `docker-compose.override.yml`:

```yaml
  app:
    environment:
      - Storage__ServiceUrl=http://minio:9000
      - Storage__AccessKey=${MINIO_ROOT_USER:-minioadmin}
      - Storage__SecretKey=${MINIO_ROOT_PASSWORD:-minioadmin}
      - Storage__BucketName=tourneygen-dev
      - Storage__Region=us-east-1
```

Add to `.env.example`:
```
# MinIO local dev (override only — not used in production)
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
```

`ForcePathStyle = true` in `AmazonS3Config` (Step 4.7) is already correct for MinIO — no extra config needed. `minio-init` retries until MinIO is ready before creating the bucket, so the app starts cleanly with no manual bucket creation steps.

### Step 1.7 — Application Dockerfile

Create `src/TourneyGen.Web/Dockerfile` from architecture section 13.5 (multi-stage build).

Gotcha: The `COPY src/ src/` layer in the Dockerfile copies the entire `src/` tree. This is correct because the `dotnet publish` must see all project source files. Ensure the `.dockerignore` file excludes `**/bin`, `**/obj`, `.git`, and `tests/`.

### Step 1.8 — Verify Scaffold Builds

Run `dotnet build TourneyGen.sln` from the repo root. The build must succeed with zero errors before proceeding. An empty solution with empty projects will build cleanly.

---

## Phase 2 — Domain & Data Model

**Depends on:** Phase 1 complete and building.

### Step 2.1 — Define Enumerations

Create in `src/TourneyGen.Domain/Enums/`:

- `EventStatus.cs` — `Draft, Active, Complete`
- `MatchOutcome.cs` — `HomeWin, AwayWin, Draw`
- `EventType.cs` — `Tournament, League`
- `TiebreakerMethod.cs` — `AutoHeadToHead, GenerateBracket, ManualSelection, CoinFlip`

These are pure C# enums with no dependencies.

### Step 2.2 — Define Domain Entities

Create in `src/TourneyGen.Domain/Entities/`. Implement each entity exactly as specified in architecture section 5.2. Key notes per entity:

**Tournament.cs:**
- `Id` is `Guid` (not `int`) — set in constructor via `Guid.NewGuid()`
- Include navigation properties but do NOT add EF Core attributes — configuration belongs in `IEntityTypeConfiguration<T>` in Infrastructure
- `Status` defaults to `EventStatus.Draft`
- `IsLocked` defaults to `false`
- `CreatedAt` set in constructor to `DateTime.UtcNow`

**League.cs:** Same pattern as Tournament. Add `TiebreakerMethod?` property.

**Participant.cs:**
- Nullable `TournamentId` and `LeagueId` — exactly one must be non-null
- The check constraint (XOR) is configured in EF Core configuration, not as a C# constraint
- 50-char `DisplayName` max enforced in EF Core configuration (not a C# attribute)

**TournamentMatch.cs:**
- `Round` is 1-indexed (Round 1 = first round, Round N = final)
- `Position` is 0-indexed within a round
- All participant FKs are nullable (null = BYE slot not yet filled, or BYE itself)
- `WinnerId` and `LoserId` nullable — null means not yet played

**LeagueFixture.cs:**
- `Leg` is 1 or 2 (double round-robin legs)
- `Outcome` is `MatchOutcome?` — null means not yet played

**SharedLink.cs:**
- `Token` is a `string(64)` — URL-safe Base64, generated by the application layer
- Nullable `TournamentId` and `LeagueId` — exactly one non-null

**ArchivedEvent.cs:**
- `PdfStorageKey` is nullable — null until PDF has been generated
- Denormalised `Name` field (copied from the event) allows display even conceptually (the hard delete cascade means this record disappears on delete too, so it only serves completed events)

Gotcha: Domain entities must have no EF Core, no System.ComponentModel.DataAnnotations, no ASP.NET references. They are plain C# classes. Validation annotations go on ViewModels (in Application/Components), not on domain entities.

### Step 2.3 — Create ApplicationUser

Create `src/TourneyGen.Infrastructure/Identity/ApplicationUser.cs`:

```csharp
public class ApplicationUser : IdentityUser
{
    public string? DisplayName { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public ICollection<Tournament> Tournaments { get; set; } = [];
    public ICollection<League> Leagues { get; set; } = [];
}
```

This is in Infrastructure (not Domain) because it extends `IdentityUser` which is an ASP.NET Identity type. Domain entities reference the `OrganiserId` string FK — they do not reference `ApplicationUser` directly, preserving Domain's independence.

Gotcha: Tournament and League entities reference `OrganiserId : string` as a FK to `ApplicationUser.Id`. The navigation property `Organiser : ApplicationUser` can be added to Tournament/League in Infrastructure via partial classes or shadow properties in EF Core configuration. Alternatively, add the navigation to the entity but import only the string type — this is a common pragmatic choice. The cleaner approach is to configure the FK relationship purely in EF Core without a navigation on the domain entity side.

### Step 2.4 — Create AppDbContext

Create `src/TourneyGen.Infrastructure/Data/AppDbContext.cs`:

- Extends `IdentityDbContext<ApplicationUser>`
- `DbSet<Tournament>`, `DbSet<League>`, `DbSet<Participant>`, `DbSet<TournamentMatch>`, `DbSet<LeagueFixture>`, `DbSet<SharedLink>`, `DbSet<ArchivedEvent>`
- Override `OnModelCreating` to call `base.OnModelCreating(builder)` first (required for Identity tables) then apply all entity configurations via `builder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly)`

### Step 2.5 — Create EF Core Entity Configurations

Create one `IEntityTypeConfiguration<T>` file per entity in `src/TourneyGen.Infrastructure/Data/Configurations/`. For each:

**TournamentConfiguration.cs:**
- Table name `Tournaments`
- `Name` column max length 200, required
- `Status` stored as string (not int) for readability: `.HasConversion<string>()`
- Index on `(OrganiserId, Status)` — named `IX_Tournaments_OrganiserId_Status`
- Index on `(OrganiserId, Status, Id)` for covering index on cap query
- FK to `ApplicationUser` with `DeleteBehavior.Cascade`
- Relationship to `Participants`, `Matches`, `SharedLink`, `ArchivedEvent`

**LeagueConfiguration.cs:** Same as Tournament. Add `TiebreakerMethod` stored as string.

**ParticipantConfiguration.cs:**
- `DisplayName` max 50, required
- Index on `TournamentId`, index on `LeagueId`
- Both FKs nullable with `DeleteBehavior.Cascade`
- Add check constraint: `CHECK ((TournamentId IS NOT NULL AND LeagueId IS NULL) OR (TournamentId IS NULL AND LeagueId IS NOT NULL))` via `.HasCheckConstraint("CK_Participant_OneEvent", ...)`

**TournamentMatchConfiguration.cs:**
- Unique constraint on `(TournamentId, Round, Position)`
- Index on `(TournamentId, Round)`
- Index on `TournamentId`
- All participant FKs use `DeleteBehavior.Restrict` (don't cascade-delete matches when participant is deleted — participants are locked; deletion of the tournament cascades differently)

Gotcha on FK cycles: MySQL/Pomelo does not support multiple cascade paths to the same table via different FKs (unlike SQL Server). `TournamentMatch` has three nullable FKs to `Participant` (Participant1, Participant2, Winner, Loser). Configure those FKs with `DeleteBehavior.Restrict` or `DeleteBehavior.SetNull` to avoid the multiple-cascade-path error.

**LeagueFixtureConfiguration.cs:**
- Unique constraint on `(LeagueId, HomeParticipantId, AwayParticipantId, Leg)`
- `Outcome` stored as string, nullable
- Three indexes as specified in architecture

**SharedLinkConfiguration.cs:**
- `Token` column max 64, required, unique index named `IX_SharedLink_Token`
- Both FKs nullable, `DeleteBehavior.Cascade` — when a tournament/league is deleted, the shared link is deleted automatically

**ArchivedEventConfiguration.cs:**
- `EventType` stored as string
- Index on `(OrganiserId, CompletedAt DESC)` — note: EF Core 8 supports descending index specification
- Both FKs nullable, `DeleteBehavior.Cascade`

### Step 2.6 — Register Infrastructure Services

Create `src/TourneyGen.Infrastructure/DependencyInjection.cs` with an extension method `AddInfrastructure(this IServiceCollection services, IConfiguration configuration)`:

```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(
        configuration.GetConnectionString("DefaultConnection"),
        ServerVersion.AutoDetect(connectionString),
        mySqlOptions => mySqlOptions.MigrationsAssembly("TourneyGen.Infrastructure")
    )
);
```

The `MigrationsAssembly` is required because `AppDbContext` is in Infrastructure but the Web project is the startup project.

> [ASP.NET Review] **Bug in the `UseMySql` call above — `connectionString` is not in scope.** Extract the connection string first:
> ```csharp
> var connectionString = configuration.GetConnectionString("DefaultConnection")
>     ?? throw new InvalidOperationException(
>         "Connection string 'DefaultConnection' not found in configuration.");
>
> services.AddDbContext<AppDbContext>(options =>
>     options.UseMySql(
>         connectionString,
>         ServerVersion.AutoDetect(connectionString),
>         mySqlOptions => mySqlOptions.MigrationsAssembly("TourneyGen.Infrastructure")
>     )
> );
> ```
> `ServerVersion.AutoDetect` opens a real TCP connection to MySQL. In development Docker Compose, the `depends_on: db: condition: service_healthy` guard ensures MySQL is ready before the app starts. In production, use a pinned version instead of `AutoDetect` to avoid the extra startup connection: `new MySqlServerVersion(new Version(8, 0, 36))`.

> [ASP.NET Review] **Connection string configuration layering.** Use this hierarchy so no credentials are ever committed:
> - `appsettings.json` — non-secret placeholder: `"DefaultConnection": "Server=localhost;Port=3306;Database=tourneygen;User=tourneygen_app;Password=CHANGEME;AllowPublicKeyRetrieval=true;SslMode=None;"`
> - **User Secrets (local dev):** `dotnet user-secrets set "ConnectionStrings:DefaultConnection" "..."` from the `src/TourneyGen.Web` directory
> - **Environment variable (Docker/prod):** `ConnectionStrings__DefaultConnection=...` (double-underscore `__` is the ASP.NET Core hierarchy separator). Docker Compose sets this via `${DB_CONNECTION_STRING}` and it overrides all `appsettings.*.json` files.

### Step 2.7 — Write and Apply Initial Migration

From the `src/TourneyGen.Infrastructure` directory (or using `--project` / `--startup-project` flags):

```
dotnet ef migrations add InitialCreate \
  --project src/TourneyGen.Infrastructure \
  --startup-project src/TourneyGen.Web \
  --output-dir Data/Migrations
```

Before running this: confirm MySQL is running via Docker Compose with `docker compose -f docker/docker-compose.yml up db -d`. Apply with:

```
dotnet ef database update \
  --project src/TourneyGen.Infrastructure \
  --startup-project src/TourneyGen.Web
```

Gotcha: EF Core requires the startup project to have `AddDbContext` registered in `Program.cs` and a design-time factory or valid `IDesignTimeDbContextFactory<AppDbContext>` to run migrations. Create `src/TourneyGen.Infrastructure/Data/AppDbContextFactory.cs` implementing `IDesignTimeDbContextFactory<AppDbContext>` to enable CLI migrations without the full web host.

Verify the migration by inspecting the generated SQL: `dotnet ef migrations script --project ... --startup-project ...`. Confirm all tables, indexes, and constraints match the data model.

> [ASP.NET Review] **`IDesignTimeDbContextFactory<AppDbContext>` implementation.** Create `src/TourneyGen.Infrastructure/Data/AppDbContextFactory.cs`:
> ```csharp
> public class AppDbContextFactory : IDesignTimeDbContextFactory<AppDbContext>
> {
>     public AppDbContext CreateDbContext(string[] args)
>     {
>         var configuration = new ConfigurationBuilder()
>             .SetBasePath(Directory.GetCurrentDirectory())
>             .AddJsonFile("appsettings.json", optional: false)
>             .AddJsonFile("appsettings.Development.json", optional: true)
>             .AddEnvironmentVariables()
>             .AddUserSecrets<AppDbContextFactory>(optional: true)
>             .Build();
>
>         var connectionString = configuration.GetConnectionString("DefaultConnection")!;
>         var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>();
>         optionsBuilder.UseMySql(
>             connectionString,
>             ServerVersion.AutoDetect(connectionString),
>             o => o.MigrationsAssembly("TourneyGen.Infrastructure"));
>
>         return new AppDbContext(optionsBuilder.Options);
>     }
> }
> ```
> Always run `dotnet ef` commands from the `src/TourneyGen.Web` directory (or pass `--startup-project src/TourneyGen.Web`) so `Directory.GetCurrentDirectory()` finds `appsettings.json` correctly.

---

## Phase 3 — Authentication

**Depends on:** Phase 2, specifically `AppDbContext` and `ApplicationUser` exist.

### Step 3.1 — Configure ASP.NET Core Identity

In `src/TourneyGen.Infrastructure/DependencyInjection.cs`, add Identity registration inside `AddInfrastructure`:

```csharp
services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.SignIn.RequireConfirmedEmail = true;
    options.Password.RequiredLength = 8;
    options.Password.RequireNonAlphanumeric = false;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```

Gotcha: Do NOT use `AddDefaultIdentity<ApplicationUser>()` if you need roles. Use `AddIdentity<ApplicationUser, IdentityRole>()`. The difference matters when adding `IdentityRole` columns in migrations.

> [ASP.NET Review] **`AddEntityFrameworkStores<AppDbContext>()` is the critical call** that wires Identity to use the Pomelo-backed `AppDbContext`. Without it, Identity uses an in-memory store and all user/role data is lost on restart. Verify the generated migration includes `AspNetUsers`, `AspNetRoles`, `AspNetUserRoles`, `AspNetUserClaims`, `AspNetUserLogins`, `AspNetUserTokens`, and `AspNetRoleClaims` tables — these come from `IdentityDbContext<ApplicationUser>` and the `base.OnModelCreating(builder)` call.

> [ASP.NET Review] **State all password options explicitly.** The block above relies on hidden defaults for `RequireUppercase`, `RequireDigit`, and `RequireLowercase` (all `true`). Add them explicitly:
> ```csharp
> options.Password.RequiredLength = 8;
> options.Password.RequireUppercase = true;
> options.Password.RequireLowercase = true;
> options.Password.RequireDigit = true;
> options.Password.RequireNonAlphanumeric = false;
> options.Password.RequiredUniqueChars = 1;
> options.Lockout.AllowedForNewUsers = true;  // Lockout applies even before email confirmation
> ```

### Step 3.2 — Configure Cookie Authentication

In `DependencyInjection.cs` or `Program.cs`, configure the application cookie per architecture section 7.4. Key setting: `SameSite = Lax` (not `Strict`) for Google OAuth callback compatibility.

Configure `LoginPath = "/account/login"` and `AccessDeniedPath = "/account/access-denied"`.

### Step 3.3 — Configure Google OAuth

Add `Microsoft.AspNetCore.Authentication.Google` to `TourneyGen.Web`. In `Program.cs`:

```csharp
builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        options.ClientId = builder.Configuration["Authentication:Google:ClientId"]!;
        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"]!;
    });
```

Required: Create a Google OAuth 2.0 Client ID in Google Cloud Console. Add `https://yourdomain.com/signin-google` as an authorised redirect URI. Store credentials in `.env` / environment variables only.

Gotcha: Google OAuth sign-ins bypass email confirmation. To implement this, handle the `OnTicketReceived` event or check `ExternalLoginInfo` on the callback and mark the email as confirmed for OAuth users in the migration/login flow.

> [ASP.NET Review] **Google OAuth callback URL for Linode deployment.** The default `CallbackPath` is `/signin-google` — this is handled entirely by ASP.NET Core middleware, not a Blazor page. Register these URIs in Google Cloud Console:
> - Development: `https://localhost:5001/signin-google`
> - Production: `https://yourdomain.com/signin-google`
>
> With `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true` (Step 1.6), the middleware correctly reconstructs the callback URI as `https://` rather than `http://`. Without it, you get a `redirect_uri_mismatch` error because the app sees its internal `http://` scheme.

> [ASP.NET Review] **Store Google credentials via `dotnet user-secrets` in development:**
> ```bash
> dotnet user-secrets set "Authentication:Google:ClientId" "<id>" --project src/TourneyGen.Web
> dotnet user-secrets set "Authentication:Google:ClientSecret" "<secret>" --project src/TourneyGen.Web
> ```
> In production, `Authentication__Google__ClientId` (double-underscore) environment variables from `docker-compose.yml` override `appsettings.json`.

> [ASP.NET Review] **Mark OAuth user email as confirmed on first login.** When creating a new `ApplicationUser` for an OAuth sign-in, set `EmailConfirmed = true` before calling `UserManager.CreateAsync`:
> ```csharp
> var user = new ApplicationUser { UserName = email, Email = email, EmailConfirmed = true };
> ```
> This prevents the confirmation email step for OAuth users whose email is already verified by Google.

### Step 3.4 — Implement Gmail SMTP Email Sender

Create `src/TourneyGen.Infrastructure/Identity/EmailSender.cs` implementing `Microsoft.AspNetCore.Identity.UI.Services.IEmailSender` (or ASP.NET Core 8's `IEmailSender<ApplicationUser>`).

Use MailKit for the SMTP implementation:

```csharp
public class GmailEmailSender : IEmailSender<ApplicationUser>
{
    // Read SmtpHost, SmtpPort, Username, Password from IOptions<EmailOptions>
    // Use MimeKit/MailKit's SmtpClient
    // Set STARTTLS on port 587
}
```

Create `EmailOptions.cs` POCO class in Infrastructure. Register via `services.Configure<EmailOptions>(configuration.GetSection("Email"))`.

Register: `services.AddTransient<IEmailSender<ApplicationUser>, GmailEmailSender>()`.

Gotcha: Gmail requires an App Password (not the account password) when 2FA is enabled. The Gmail account must have 2FA enabled. App Passwords are 16-character strings generated in Google Account security settings.

> [ASP.NET Review] **Complete `EmailOptions` POCO.** Define in `src/TourneyGen.Infrastructure/Identity/EmailOptions.cs`:
> ```csharp
> public class EmailOptions
> {
>     public string SmtpHost { get; set; } = "smtp.gmail.com";
>     public int SmtpPort { get; set; } = 587;
>     public string Username { get; set; } = string.Empty;    // Full Gmail address
>     public string Password { get; set; } = string.Empty;    // 16-char App Password
>     public string SenderName { get; set; } = "TourneyGen";
>     public string SenderAddress { get; set; } = string.Empty;
> }
> ```
> The MailKit `SmtpClient` must use `SecureSocketOptions.StartTls` on port 587 — NOT `SslOnConnect` (that is port 465). Gmail's port 587 requires STARTTLS (explicit upgrade). Minimal correct send pattern:
> ```csharp
> using var client = new MailKit.Net.Smtp.SmtpClient();
> await client.ConnectAsync(_options.SmtpHost, _options.SmtpPort,
>     MailKit.Security.SecureSocketOptions.StartTls, cancellationToken);
> await client.AuthenticateAsync(_options.Username, _options.Password, cancellationToken);
> await client.SendAsync(message, cancellationToken);
> await client.DisconnectAsync(true, cancellationToken);
> ```
> Do NOT use `System.Net.Mail.SmtpClient` — it is marked obsolete and has STARTTLS reliability issues on Linux.

> [ASP.NET Review] **Gmail App Password pre-deployment steps:**
> 1. Enable 2-Step Verification on the Gmail account.
> 2. Google Account → Security → 2-Step Verification → App passwords → generate for "Mail".
> 3. Dev: `dotnet user-secrets set "Email:Password" "<16-char-password>" --project src/TourneyGen.Web`
> 4. Prod: `Email__Password=<16-char-password>` in the `.env` file on the VPS.
> The regular Gmail password returns `535 5.7.8 Username and Password not accepted` — it will NOT work.

> [ASP.NET Review] **Use `IEmailSender<ApplicationUser>` (the .NET 8 generic interface).** Register `GmailEmailSender` against `IEmailSender<ApplicationUser>` from `Microsoft.AspNetCore.Identity` — this is the interface Identity's built-in confirmation and password reset flows call in .NET 8. The non-generic `IEmailSender` from `Microsoft.AspNetCore.Identity.UI` is the .NET 6/7 pattern.

### Step 3.5 — Create Auth Razor Pages / Blazor Pages

Create the Blazor SSR pages in `src/TourneyGen.Web/Components/Pages/Account/`:

- `Login.razor` — email/password form + "Sign in with Google" button
- `Register.razor` — registration form (email, password, confirm password)
- `EmailConfirmation.razor` — handles `?userId=&code=` query params, confirms email
- `ForgotPassword.razor` — email input, triggers password reset email
- `ResetPassword.razor` — new password form, handles `?email=&code=` query params
- `AccessDenied.razor`

All use Static SSR with `@attribute [AllowAnonymous]`. Forms submit to `@page` routes using Blazor's enhanced form handling (`<EditForm ... FormName="login" ...>`).

Gotcha: In Blazor Web Apps with static SSR, form submissions require the `@rendermode` to be either SSR or to have the form submit to an explicit route. Use Blazor 8's `[SupplyParameterFromForm]` attribute for model binding on SSR form posts.

### Step 3.6 — Add Rate Limiting to Auth Routes

In `Program.cs`, add rate limiting middleware per architecture section 11.2:

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
// ...
app.UseRateLimiter();
```

Apply the `[EnableRateLimiting("auth")]` attribute to Login and Register pages or map them explicitly.

### Step 3.7 — Implement Guest Session State

Create `src/TourneyGen.Application/Identity/GuestTournamentState.cs`:

```csharp
public record GuestTournamentState(
    List<string> ParticipantNames,
    int[][] BracketSeeding,
    List<GuestMatchResult> Results
);

public record GuestMatchResult(int Round, int Position, int WinnerIndex);
```

This record lives in Application (not Domain) because it is a transient application state, not a persistent domain entity.

The state is held in the Blazor `InteractiveServer` component's field — it is never serialised to any storage. The component owns its lifecycle.

### Step 3.8 — Implement GuestMigrationService

Create `src/TourneyGen.Application/Identity/IGuestMigrationService.cs`:

```csharp
public interface IGuestMigrationService
{
    Task<Tournament> MigrateToAccountAsync(
        GuestTournamentState guestState,
        string organiserId,
        CancellationToken ct = default);
}
```

Create `src/TourneyGen.Infrastructure/Identity/GuestMigrationService.cs` implementing this interface. It:
1. Creates a `Tournament` record (Status = Active or Complete based on guest state)
2. Creates `Participant` rows for each name in `guestState.ParticipantNames`
3. Reconstructs `TournamentMatch` rows from `guestState.BracketSeeding` and `guestState.Results`
4. Saves all within a single `DbContext.SaveChangesAsync()` call (one transaction)

The component calls this service after the user completes the sign-in flow, passing the in-memory `GuestTournamentState`. Navigation then redirects to the newly created tournament page.

Gotcha: The migration must check the active-event cap (Step 4.1). A guest with one in-progress tournament migrating while the account already has 5 active events should get an error — the UI should warn about this before prompting login.

---

## Phase 4 — Application Services

**Depends on:** Phase 2 entities and DbContext, Phase 3 Identity.

### Step 4.1 — Implement TournamentService

Create `src/TourneyGen.Application/Tournaments/ITournamentService.cs` with all methods from architecture section 6.2.

Create `src/TourneyGen.Infrastructure/Tournaments/TournamentService.cs` (or keep in Application if it only uses interfaces — but since it needs EF Core directly for queries, it lives in Infrastructure).

Wait — Architecture section 6.1 says services are in `TourneyGen.Application` and implementations in `Infrastructure`. Since Application services in this architecture directly use `AppDbContext` (there are no repository interfaces), the implementations live in Infrastructure. The interfaces in Application reference only domain types and no EF Core types.

Actually re-reading the architecture: there's no repository layer. The `TournamentService` in Application can reference `AppDbContext` via a constructor parameter only if Infrastructure provides it — but Application must not reference Infrastructure. The resolution is: `TournamentService` implementations live in Infrastructure (which references Application), and the `ITournamentService` interface + ViewModels live in Application.

So file locations:
- `src/TourneyGen.Application/Tournaments/ITournamentService.cs` — interface
- `src/TourneyGen.Application/Tournaments/BracketViewModel.cs` — ViewModel record
- `src/TourneyGen.Infrastructure/Services/TournamentService.cs` — implementation

Key implementation details for each method:

**`CreateAsync`:**
- Check active event cap: `COUNT(*) WHERE OrganiserId = x AND Status != Complete` across both Tournaments and Leagues tables. If total >= 5, throw `ActiveEventLimitException`.
- Validate `participantCount` is one of {4, 8, 16, 32, 64}.
- Create `Tournament` with `Status = Draft`, `IsLocked = false`.
- Wrap in a transaction to prevent race condition on the cap check.

> [ASP.NET Review] **Use `IsolationLevel.Serializable` for the cap-check transaction.** MySQL's default `RepeatableRead` does not prevent phantom inserts — two concurrent requests can both read a count of 4, both conclude the cap is not exceeded, and both insert, resulting in 6 active events. Use `Serializable` to acquire range locks:
> ```csharp
> await using var transaction = await _db.Database.BeginTransactionAsync(
>     System.Data.IsolationLevel.Serializable, cancellationToken);
> try
> {
>     var activeTournaments = await _db.Tournaments
>         .CountAsync(t => t.OrganiserId == organiserId && t.Status != EventStatus.Complete, cancellationToken);
>     var activeLeagues = await _db.Leagues
>         .CountAsync(l => l.OrganiserId == organiserId && l.Status != EventStatus.Complete, cancellationToken);
>     if (activeTournaments + activeLeagues >= 5)
>         throw new ActiveEventLimitException(organiserId);
>     // ... create tournament ...
>     await _db.SaveChangesAsync(cancellationToken);
>     await transaction.CommitAsync(cancellationToken);
> }
> catch { await transaction.RollbackAsync(cancellationToken); throw; }
> ```
> Define `ActiveEventLimitException` in `src/TourneyGen.Application/Tournaments/ActiveEventLimitException.cs` (not in Domain, not in Infrastructure). It is referenced by both `TournamentService` and `LeagueService` in Infrastructure, both of which depend on Application.

**`AddParticipantAsync`:**
- Validate tournament `IsLocked == false` (otherwise throw `TournamentLockedException`).
- Validate `displayName.Length <= 50`.
- Validate current participant count < `tournament.ParticipantCount`.

**`ShuffleSeeding` (not async — works on in-memory state):**
- Takes a list of participant IDs and returns them in a new random order.
- The shuffle is NOT persisted until `StartAsync` is called.
- Uses `RandomNumberGenerator`-backed shuffle (Fisher-Yates), not `Random`, for security.

**`StartAsync`:**
- Validates participant count equals `tournament.ParticipantCount` (all slots filled).
- Sets `IsLocked = true`, `Status = Active`, `StartedAt = UtcNow`.
- Persists the final seeding order as `TournamentMatch` rows for Round 1 (all `n/2` first-round matches, with `Participant1Id` and `Participant2Id` set from the seeding array).
- Creates empty match slots for all subsequent rounds (Round 2 through log2(n)) with null participant FKs — they fill in as rounds progress.

Gotcha: Round count = `log2(participantCount)`. For 64 participants: 6 rounds. Pre-create all match slots even for later rounds — this makes bracket display and result recording simpler (no dynamic row creation needed).

**`RecordResultAsync`:**
- Fetches the match, validates both participants are non-null.
- Sets `WinnerId`, `LoserId`, `PlayedAt`.
- Advances winner to next round: find the match in `Round + 1` at `Position / 2`, set `Participant1Id` or `Participant2Id` based on `Position % 2`.
- Checks if this was the final match: if so, set `Tournament.Status = Complete`, `CompletedAt = UtcNow`, and create an `ArchivedEvent` record.

**`EditResultAsync`:**
- Checks if any later-round matches have a `WinnerId` set. If yes and `confirmCascade == false`, throw `CascadeConfirmationRequiredException` (the UI catches this and shows the warning modal).
- If `confirmCascade == true` (or no downstream results): clear `WinnerId`/`LoserId` for all affected downstream matches, then re-apply the new result.
- "Affected downstream" = all matches in rounds > current match's round where `Participant1Id` or `Participant2Id` equals the previous winner.

### Step 4.2 — Implement BracketEngine

Create `src/TourneyGen.Application/Tournaments/BracketEngine.cs` — a static helper class with:
- `Shuffle<T>(IList<T> list)` — Fisher-Yates shuffle
- `BuildBracketViewModel(Tournament t, IList<Participant> participants, IList<TournamentMatch> matches) -> BracketViewModel`

`BracketViewModel` contains:
```csharp
public record BracketViewModel(
    Guid TournamentId,
    string Name,
    int RoundCount,
    IReadOnlyList<RoundViewModel> Rounds
);

public record RoundViewModel(int RoundNumber, IReadOnlyList<MatchViewModel> Matches);

public record MatchViewModel(
    Guid MatchId,
    string? Participant1Name,
    string? Participant2Name,
    string? WinnerName,
    bool IsComplete,
    bool IsBye
);
```

`GetBracketAsync` in TournamentService calls `BracketEngine.BuildBracketViewModel` after fetching data.

### Step 4.3 — Implement LeagueService

Same structure as TournamentService. Key method details:

**`StartAsync`:**
- Validates participant count is 4–64.
- Calls `FixtureGenerator.GenerateFixtures(participants)` to produce all `LeagueFixture` rows.
- Sets `IsLocked = true`, `Status = Active`.

**`RecordFixtureResultAsync` / `EditFixtureResultAsync`:**
- Set `Outcome` and `PlayedAt` on the fixture.
- After edit: check if all fixtures now have an `Outcome`. If yes, compute standings and check for a first-place tie. If tied and no `TiebreakerMethod` has been set, do NOT auto-complete — the UI will detect this state and show the tiebreaker modal.

**`GetStandingsAsync`:**
- Calls `StandingsCalculator.Calculate(fixtures, participants) -> IReadOnlyList<StandingRow>`
- Does not query DB for standings (no standings table) — computes in-memory from fixture data.

**`ResolveTiebreakerAsync`:**
- Handles each `TiebreakerMethod` case.
- For `AutoHeadToHead`: queries head-to-head fixtures between the tied participants, computes a mini-standings, identifies a winner.
- For `GenerateBracket`: creates a new in-progress bracket (internal use, not a separate tournament entity) and returns it for UI display. Gotcha: Decide whether a tiebreaker bracket is a separate entity or embedded state. For v1 simplicity, embed it as additional `TournamentMatch`-like records tagged with the league ID, or use a separate lightweight in-memory flow. The PRD says "generate a small single-elimination bracket for the tied participants" — this is a UI flow, not necessarily persisted as a separate tournament. Use a flag/state on the League entity or create a lightweight embedded sub-entity.
- For `CoinFlip`: `RandomNumberGenerator`-backed selection from tied participants.
- For `ManualSelection`: accepts `manualWinnerId`, validates it is one of the tied first-place participants.
- After resolution: set `League.TiebreakerMethod`, check if all fixtures have results, if yes set `Status = Complete`, create `ArchivedEvent`.

### Step 4.4 — Implement FixtureGenerator

Create `src/TourneyGen.Application/Leagues/FixtureGenerator.cs`:

```csharp
public static class FixtureGenerator
{
    public static IReadOnlyList<(Guid HomeId, Guid AwayId, int Leg)>
        GenerateFixtures(IReadOnlyList<Participant> participants)
```

Use the round-robin scheduling algorithm (circle method / polygon rotation algorithm) to generate a balanced schedule. For `n` participants (add a dummy "BYE" participant if n is odd):

1. Fix participant at position 0, rotate others around positions 1..n-1.
2. For each round, produce `n/2` pairings.
3. Total rounds = `n - 1` (even n) or `n` (odd n).
4. Repeat for Leg 2 with home/away reversed.

This produces `n*(n-1)` fixtures for even n, and `n*(n-1)` for odd n (BYE fixtures are excluded from the output).

Gotcha: With 64 participants, this generates 64*63 = 4,032 fixtures (double round-robin). Inserting 4,032 rows in a single `SaveChangesAsync` call is fine at this scale but use `AddRange` for efficiency.

### Step 4.5 — Implement StandingsCalculator

Create `src/TourneyGen.Application/Leagues/StandingsCalculator.cs`:

```csharp
public static class StandingsCalculator
{
    public static IReadOnlyList<StandingRow> Calculate(
        IReadOnlyList<LeagueFixture> fixtures,
        IReadOnlyList<Participant> participants)
```

`StandingRow`:
```csharp
public record StandingRow(
    Guid ParticipantId,
    string DisplayName,
    int Rank,         // shared rank for tied participants
    int Points,
    int Played,
    int Won,
    int Drawn,
    int Lost
);
```

Algorithm:
1. Accumulate stats per participant from all fixtures with non-null `Outcome`.
2. Sort by `Points DESC`.
3. Assign rank with tie-sharing: participants with equal points share the same rank number. Rank increments by the number of tied participants (e.g., 3-way tie at rank 1 → next rank is 4, not 2).

### Step 4.6 — Implement SharingService

Create interfaces in `src/TourneyGen.Application/Sharing/` and implementation in Infrastructure.

`GenerateLinkAsync`:
- Check that the event belongs to the calling organiser (ownership validation).
- Check if a `SharedLink` already exists for the event — return existing if so (idempotent).
- Generate token: `Base64UrlTextEncoder.Encode(RandomNumberGenerator.GetBytes(32))`.
- Create and persist `SharedLink` row.
- Return the full URL: `$"{baseUrl}/share/{token}"`.

`ResolveTokenAsync`:
- Query `SharedLink` by token (uses unique index — O(1)).
- If not found: return null (uniform response for invalid + deleted tokens).
- If found: load the associated tournament or league + all participants + all matches/fixtures.
- Compute "last updated" timestamp as the max `PlayedAt` across all matches/fixtures, falling back to `StartedAt`.
- Return a `SharedEventViewModel` containing all data needed for read-only rendering.

### Step 4.7 — Implement ExportService

Create interfaces in `src/TourneyGen.Application/Export/` and implementation in Infrastructure.

`GeneratePdfAsync`:
- Load `ArchivedEvent`, then load the full tournament/league data.
- Build QuestPDF document — see architecture section 10.3 for content spec.
- Write PDF to `MemoryStream`.
- Upload to S3 at key `pdfs/{organiserId}/{archivedEventId}.pdf`.
- Update `ArchivedEvent.PdfStorageKey` in the DB.
- Return the storage key.

`GetDownloadUrlAsync`:
- Generate presigned S3 URL with 1-hour TTL.
- Do NOT re-generate the PDF if `PdfStorageKey` is already set.

S3 client configuration: inject `IAmazonS3` via `IOptions<StorageOptions>`. Register in `DependencyInjection.cs`:

```csharp
services.Configure<StorageOptions>(configuration.GetSection("Storage"));
services.AddSingleton<IAmazonS3>(sp =>
{
    var opts = sp.GetRequiredService<IOptions<StorageOptions>>().Value;
    var config = new AmazonS3Config
    {
        ServiceURL = opts.ServiceUrl,
        ForcePathStyle = true  // Required for Linode Object Storage
    };
    return new AmazonS3Client(opts.AccessKey, opts.SecretKey, config);
});
```

Gotcha: Linode Object Storage requires `ForcePathStyle = true` and the region-specific `ServiceURL`. The SDK defaults to AWS endpoints if not overridden.

> [ASP.NET Review] **QuestPDF license must be set in `Program.cs` before `builder.Build()`.** QuestPDF throws a `QuestPDFException` at runtime if the license is not configured before the first `Document.Create()` call:
> ```csharp
> QuestPDF.Settings.License = QuestPDF.Infrastructure.LicenseType.Community;
> ```
> Place this in `Program.cs` before `builder.Build()`. The Community license is free for applications under $1M USD annual revenue. Do NOT place this call inside a service method or static constructor — it must run before any QuestPDF type is used.

> [ASP.NET Review] **`StorageOptions` needs a `Region` field.** The AWS SDK signs requests with a region even for S3-compatible services. Without `AuthenticationRegion`, Linode rejects requests with `SignatureDoesNotMatch`. Update:
> ```csharp
> public class StorageOptions
> {
>     public string ServiceUrl { get; set; } = string.Empty;
>     public string BucketName { get; set; } = string.Empty;
>     public string AccessKey { get; set; } = string.Empty;
>     public string SecretKey { get; set; } = string.Empty;
>     public string Region { get; set; } = "us-east-1";
> }
> ```
> Update the `AmazonS3Client` construction:
> ```csharp
> var credentials = new Amazon.Runtime.BasicAWSCredentials(opts.AccessKey, opts.SecretKey);
> var config = new AmazonS3Config
> {
>     ServiceURL = opts.ServiceUrl,
>     ForcePathStyle = true,
>     AuthenticationRegion = opts.Region
> };
> return new AmazonS3Client(credentials, config);
> ```

> [ASP.NET Review] **`GetPreSignedURL` is synchronous — do not `await` it.** There is no `GetPreSignedURLAsync` in AWSSDK.S3. URL generation is pure HMAC computation with no I/O:
> ```csharp
> public Task<string> GetDownloadUrlAsync(string storageKey, CancellationToken ct = default)
> {
>     var request = new GetPreSignedUrlRequest
>     {
>         BucketName = _options.BucketName,
>         Key = storageKey,
>         Expires = DateTime.UtcNow.AddHours(1),
>         Verb = HttpVerb.GET
>     };
>     return Task.FromResult(_s3Client.GetPreSignedURL(request));
> }
> ```

> [ASP.NET Review] **Store S3 credentials via user-secrets in development.** Never put Linode access keys in `appsettings.json`:
> ```bash
> dotnet user-secrets set "Storage:AccessKey" "<key>" --project src/TourneyGen.Web
> dotnet user-secrets set "Storage:SecretKey" "<secret>" --project src/TourneyGen.Web
> ```

### Step 4.8 — Register All Services

In `DependencyInjection.cs`, register all service implementations:

```csharp
services.AddScoped<ITournamentService, TournamentService>();
services.AddScoped<ILeagueService, LeagueService>();
services.AddScoped<ISharingService, SharingService>();
services.AddScoped<IExportService, PdfExportService>();
services.AddScoped<IGuestMigrationService, GuestMigrationService>();
```

Use `AddScoped` (not `AddSingleton`) because services use `AppDbContext` which is scoped to the HTTP request.

> [ASP.NET Review] **Complete DI registration checklist for `DependencyInjection.cs`.** All of the following must be present to avoid missing-service runtime errors:
> ```csharp
> // EF Core
> services.AddDbContextFactory<AppDbContext>(...);  // For InteractiveServer services
> services.AddDbContext<AppDbContext>(...);          // For SSR request-scoped usage and migrations
>
> // Identity (must follow AddDbContext)
> services.AddIdentity<ApplicationUser, IdentityRole>(...)
>     .AddEntityFrameworkStores<AppDbContext>()
>     .AddDefaultTokenProviders();
>
> // Cookie (must follow AddIdentity)
> services.ConfigureApplicationCookie(...);
>
> // Email (Transient — new SmtpClient per send is correct for MailKit)
> services.Configure<EmailOptions>(configuration.GetSection("Email"));
> services.AddTransient<IEmailSender<ApplicationUser>, GmailEmailSender>();
>
> // Storage (Singleton — AmazonS3Client is thread-safe, no DbContext dependency)
> services.Configure<StorageOptions>(configuration.GetSection("Storage"));
> services.AddSingleton<IAmazonS3>(sp => { ... });
>
> // Application services (Scoped — all depend on DbContext which is Scoped)
> services.AddScoped<ITournamentService, TournamentService>();
> services.AddScoped<ILeagueService, LeagueService>();
> services.AddScoped<ISharingService, SharingService>();
> services.AddScoped<IExportService, PdfExportService>();
> services.AddScoped<IGuestMigrationService, GuestMigrationService>();
> ```
> `IAmazonS3` is `Singleton` because `AmazonS3Client` is thread-safe and stateless. `AddRateLimiter` belongs in `Program.cs` (web concern), not here.

> [Blazor Review] **Critical: Scoped service lifetime differs between Static SSR and InteractiveServer.** In Static SSR, a scoped service lives for the duration of one HTTP request — identical to Razor Pages behaviour. In `InteractiveServer`, a scoped service lives for the duration of the **SignalR circuit**, not a single request. This means a single `AppDbContext` instance is reused across all interactions in an interactive component's lifetime. EF Core's `DbContext` is not designed to be long-lived — it accumulates change tracker entries over time. To avoid stale data and memory growth:
> - In `InteractiveServer` components, do NOT hold injected services across multiple user interactions. Call the service in response to each user action and let the service create a fresh scoped `DbContext` via `IDbContextFactory<AppDbContext>`.
> - Register the DbContext with `AddDbContextFactory<AppDbContext>` (in addition to or instead of `AddDbContext`) in `DependencyInjection.cs`. Services in Infrastructure then inject `IDbContextFactory<AppDbContext>` and call `await _dbContextFactory.CreateDbContextAsync()` inside each method, disposing the context after use.
> - This pattern is the official Microsoft recommendation for Blazor Server: [EF Core with Blazor Server](https://learn.microsoft.com/en-us/aspnet/core/blazor/blazor-ef-core).
>
> Update `DependencyInjection.cs` to register both:
> ```csharp
> services.AddDbContextFactory<AppDbContext>(options =>
>     options.UseMySql(connectionString, serverVersion,
>         mySqlOptions => mySqlOptions.MigrationsAssembly("TourneyGen.Infrastructure")));
> // AddDbContext is still needed for EF Core migrations tooling and SSR request-scoped usage:
> services.AddDbContext<AppDbContext>(options =>
>     options.UseMySql(connectionString, serverVersion,
>         mySqlOptions => mySqlOptions.MigrationsAssembly("TourneyGen.Infrastructure")));
> ```
> Service implementations in Infrastructure inject `IDbContextFactory<AppDbContext>` and create/dispose contexts per operation.

---

## Phase 5 — Blazor UI: Core Pages

**Depends on:** Phase 3 for auth pages infrastructure, Phase 4 for service contracts.

### Step 5.1 — Configure Program.cs

In `src/TourneyGen.Web/Program.cs`, assemble the full pipeline:

1. `builder.Services.AddRazorComponents().AddInteractiveServerComponents()` — note this is a **single chained call**, not two separate calls. `AddRazorComponents()` returns `IRazorComponentsBuilder`; `.AddInteractiveServerComponents()` is called on that return value. Do NOT call `AddInteractiveServerComponents()` independently on `builder.Services`.
2. `builder.Services.AddInfrastructure(builder.Configuration)` (from Phase 2/3)
3. `builder.Services.AddCascadingAuthenticationState()` — required for `.NET 8+` Blazor Web App. This registers the `AuthenticationStateProvider` cascade so `AuthorizeView` and `[Authorize]` work in **both SSR and InteractiveServer components** without any additional wiring.
4. `builder.Services.AddRateLimiter(...)` (Phase 3)
5. Build the app.
6. Do NOT call `app.UseHttpsRedirection()` — Nginx handles TLS termination (architecture section 11.2). Call `app.UseForwardedHeaders()` instead with `ForwardedHeadersOptions` set to forward `X-Forwarded-Proto` and `X-Forwarded-For`.
7. `app.UseAuthentication()`, `app.UseAuthorization()`
8. `app.UseRateLimiter()`
9. `app.MapRazorComponents<App>().AddInteractiveServerRenderMode()` — note this also uses method chaining on the return value of `MapRazorComponents<App>()`. Do NOT call `AddInteractiveServerRenderMode()` separately.
10. Dev-only: `await db.Database.MigrateAsync()` (per architecture section 13.3)

> [Blazor Review] `AddCascadingAuthenticationState()` (introduced in .NET 8) eliminates the need to wrap `<Router>` or `<Routes>` in a manually added `<CascadingAuthenticationState>` tag in `App.razor`. In .NET 8+, calling this service registration method is sufficient — do not add `<CascadingAuthenticationState>` as a component wrapper in `App.razor` or `Routes.razor` as this would be redundant and can cause double-cascading issues.

> [Blazor Review] The correct `Program.cs` skeleton for the service registrations and middleware pipeline is:
> ```csharp
> // --- Service registrations ---
> builder.Services.AddRazorComponents()
>     .AddInteractiveServerComponents();
>
> builder.Services.AddCascadingAuthenticationState();
> builder.Services.AddInfrastructure(builder.Configuration);
> builder.Services.AddRateLimiter(/* ... */);
>
> // --- Middleware pipeline ---
> var app = builder.Build();
>
> app.UseForwardedHeaders(new ForwardedHeadersOptions
> {
>     ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
> });
> app.UseStaticFiles();
> app.UseAntiforgery();           // MUST come before MapRazorComponents
> app.UseRateLimiter();
> app.UseAuthentication();
> app.UseAuthorization();
>
> app.MapRazorComponents<App>()
>     .AddInteractiveServerRenderMode();
> ```
> `app.UseAntiforgery()` is required for Blazor SSR form handling (the `EditForm` antiforgery token check). It must be placed between `UseStaticFiles` and `MapRazorComponents`. Omitting it causes a 400 Bad Request on all SSR form posts.

Gotcha: `UseForwardedHeaders` must be called BEFORE `UseAuthentication` and `UseAuthorization`. Configure it with `KnownProxies` set to the Nginx container's internal IP (or `IPAddress.Any`) only in production behind a trusted proxy. In development (no Nginx), you can omit `UseForwardedHeaders` entirely.

### Step 5.2 — Configure App.razor and Routes.razor

`App.razor`:
- Include `<link>` tags for Bootstrap 5 CSS (from CDN or wwwroot)
- Include the RCL component-scoped CSS bundle: `<link rel="stylesheet" href="_content/TourneyGen.Components/TourneyGen.Components.styles.css" />`
- Global Blazor scripts: `<script src="_framework/blazor.web.js"></script>` — this single script handles both SSR enhanced navigation and InteractiveServer circuit negotiation.

`Routes.razor`:
- Use `<AuthorizeRouteView>` with `DefaultLayout` set to `MainLayout`
- Add `NotAuthorized` template that redirects to `/account/login`
- Do NOT wrap `Routes.razor` content in `<CascadingAuthenticationState>` — this is handled by `AddCascadingAuthenticationState()` in `Program.cs` (see Step 5.1).

> [Blazor Review] The `Routes.razor` file must NOT have any `@rendermode` directive applied to it. A render mode on `Routes.razor` would make the entire router interactive, contradicting the per-page render mode strategy. The correct pattern is: `Routes.razor` stays as Static SSR (no directive), and individual page components in `TourneyGen.Components` declare `@rendermode InteractiveServer` on themselves.
>
> The minimal correct `Routes.razor` for this architecture:
> ```razor
> <Router AppAssembly="@typeof(App).Assembly"
>         AdditionalAssemblies="@(new[] { typeof(TourneyGen.Components.Tournament.TournamentSetup).Assembly })">
>     <Found Context="routeData">
>         <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)">
>             <NotAuthorized>
>                 <RedirectToLogin />
>             </NotAuthorized>
>         </AuthorizeRouteView>
>         <FocusOnNavigate RouteData="@routeData" Selector="h1" />
>     </Found>
>     <NotFound>
>         <PageTitle>Not found</PageTitle>
>         <LayoutView Layout="@typeof(MainLayout)">
>             <p role="alert">Sorry, there's nothing at this address.</p>
>         </LayoutView>
>     </NotFound>
> </Router>
> ```
> The `AdditionalAssemblies` parameter is **required** because the `@page` route directives for bracket, fixture, and other pages live in the `TourneyGen.Components` RCL assembly — not in `TourneyGen.Web`. Without `AdditionalAssemblies`, those routes are invisible to the router and will always render the `<NotFound>` fallback.
>
> Create `src/TourneyGen.Web/Components/RedirectToLogin.razor` as a simple SSR component:
> ```razor
> @inject NavigationManager Navigation
> @code {
>     protected override void OnInitialized()
>         => Navigation.NavigateTo(
>             $"/account/login?returnUrl={Uri.EscapeDataString(Navigation.Uri)}",
>             forceLoad: false);
> }
> ```

### Step 5.3 — Create MainLayout

`src/TourneyGen.Web/Components/Layout/MainLayout.razor`:
- Bootstrap 5 navbar with the TourneyGen logo/name on the left
- `AuthorizeView` in the top-right:
  - If authenticated: show user name + Logout link
  - If not authenticated: show Login button
- `@Body` content area

### Step 5.4 — Landing Page

`src/TourneyGen.Web/Components/Pages/Index.razor` (Static SSR, no `@rendermode`):
- Two large CTA buttons: "Create Tournament" and "Create League"
- Login button top-right (handled by MainLayout)
- "Create League" button: wrapped in `<AuthorizeView>` — if not authenticated, shows a tooltip or redirects to `/account/login?returnUrl=/league/create`
- "Create Tournament" button: always active — navigates to `/tournament/create` (guest flow if not logged in)

> [Blazor Review] On a Static SSR page, `<AuthorizeView>` works correctly because `AddCascadingAuthenticationState()` provides the `Task<AuthenticationState>` cascade even during SSR rendering. No additional configuration is needed. However, do NOT use `[CascadingParameter] Task<AuthenticationState> AuthStateTask` directly in SSR page components — use `<AuthorizeView>` in markup or inject `IHttpContextAccessor` and read `HttpContext.User` for imperative checks. The `Task<AuthenticationState>` parameter pattern is for interactive components; on SSR pages, the synchronous `HttpContext.User` is already resolved before the component renders.

### Step 5.5 — Account Dashboard

`src/TourneyGen.Web/Components/Pages/Dashboard.razor` (Static SSR, `@attribute [Authorize]`):
- Inject `ITournamentService` and `ILeagueService`
- Inject `[CascadingParameter] HttpContext HttpContext` — use `HttpContext.User.FindFirstValue(ClaimTypes.NameIdentifier)` to get the `organiserId` for service calls. Do NOT inject `AuthenticationStateProvider` on a Static SSR page — it is not available outside an interactive render context.
- On render: fetch active tournaments and leagues for the current organiser
- Display two sections: "Active Events" and "Archived Events"
- Each event card: name, type badge (Tournament/League), status, navigate-on-click
- "Create New Event" button (enforces 5-event cap in service — the button shows as disabled with tooltip if cap reached)

Use `@attribute [StreamRendering(true)]` to show a loading state while service data fetches. This is the .NET 8 Blazor Static SSR streaming pattern. `[StreamRendering]` is only valid on Static SSR components (no `@rendermode` directive); applying it to an interactive component is a compile error.

> [Blazor Review] Getting the authenticated user's ID in Static SSR pages vs. InteractiveServer components requires different approaches:
> - **Static SSR pages** (Dashboard, landing, auth pages): Use `[CascadingParameter] HttpContext HttpContext` and read `HttpContext.User`. `HttpContext` is available as a cascading parameter on SSR pages.
> - **InteractiveServer components** (BracketView, FixtureList, etc.): Inject `AuthenticationStateProvider` and call `await AuthenticationStateProvider.GetAuthenticationStateAsync()`. Alternatively, declare `[CascadingParameter] private Task<AuthenticationState> AuthenticationStateTask { get; set; }` — this works because `AddCascadingAuthenticationState()` provides this cascade to all components including interactive ones.
> - **Do NOT use `[CascadingParameter] HttpContext`** in InteractiveServer components — `HttpContext` is null after the initial SSR prerender phase ends and the SignalR circuit takes over.
> - The RCL components (`TournamentSetup`, `BracketView`, etc.) should use `[CascadingParameter] Task<AuthenticationState>` for auth state access since the same component may eventually run in MAUI context where `HttpContext` is unavailable.

---

## Phase 6 — Blazor UI: Tournament Flow

**Depends on:** Phase 4 (TournamentService, BracketEngine) and Phase 5 (layouts, auth).

### Step 6.1 — Create Tournament Page

`src/TourneyGen.Components/Tournament/TournamentSetup.razor` (`@rendermode InteractiveServer`):

State:
- `string _name` — tournament name input
- `int _participantCount` — dropdown: 4, 8, 16, 32, 64
- `List<string> _participants` — names entered so far
- `string _newParticipantInput` — text input for next name
- `bool _isGuest` — injected as `[Parameter]`

UI:
- Step 1: Name the tournament + select participant count. "Next" advances to step 2.
- Step 2: Participant entry form. Input + "Add" button. List of added names below. "Remove" button per name.
- Validate: name ≤ 50 chars. Warn (not block) on duplicates.
- "Proceed to Seeding" button — enabled only when `_participants.Count == _participantCount`.

> [Blazor Review] **Use `EditForm` with `DataAnnotationsValidator` for participant entry.** Do not use plain HTML `<form>` or direct `@oninput`/`@onchange` wiring for the participant name input. The correct pattern for an `InteractiveServer` component:
> ```razor
> <EditForm Model="@_addParticipantModel" OnValidSubmit="@HandleAddParticipant">
>     <DataAnnotationsValidator />
>     <InputText @bind-Value="_addParticipantModel.Name" class="form-control" />
>     <ValidationMessage For="@(() => _addParticipantModel.Name)" />
>     <button type="submit" class="btn btn-primary">Add</button>
> </EditForm>
> ```
> Where `_addParticipantModel` is a ViewModel class (in `TourneyGen.Application` or `TourneyGen.Components`) with `[Required, MaxLength(50)]` annotations. Validation runs server-side in the SignalR circuit — there is no client-side WASM validation module in this architecture (unlike Blazor WebAssembly). `DataAnnotationsValidator` works identically in both modes. Do NOT create separate client-side and server-side validation paths.

> [Blazor Review] **`OnInitializedAsync` vs prerendering.** `TournamentSetup` is `@rendermode InteractiveServer`, which by default prerenders once as Static SSR before the SignalR circuit connects. This means `OnInitializedAsync` runs **twice**: once during SSR prerender and once when the interactive circuit starts. For this component, state initialisation in `OnInitializedAsync` is safe because the state is empty lists and primitive defaults — there are no DB calls or external service calls in `OnInitializedAsync`. This remains correct. However, if future versions of this component load existing tournament data from a service in `OnInitializedAsync`, that service call will execute twice. Use `PersistentComponentState` or check `firstRender` in `OnAfterRenderAsync` for data loads that must not double-execute.

> [Blazor Review] **`NavigationManager` for post-action navigation.** After a successful "Proceed to Seeding" action (organiser flow: `StartAsync` completed; guest flow: seeding state set), navigate to the bracket seeding page using `NavigationManager.NavigateTo`:
> ```csharp
> @inject NavigationManager Navigation
> // ...
> private async Task HandleConfirmSetup()
> {
>     // ... service call or state mutation ...
>     Navigation.NavigateTo($"/tournament/{_tournamentId}/seeding");
> }
> ```
> Do NOT use `<a href>` tag navigation for post-form actions in InteractiveServer components — the component state would be discarded. Use `NavigationManager.NavigateTo` from event handlers. The `forceLoad: false` default (enhanced navigation) is correct here.

Host page: `src/TourneyGen.Web/Components/Pages/Tournament/Create.razor` — this is the thin SSR host page. It declares `@page "/tournament/create"` and renders `<TournamentSetup IsGuest="@(!isAuthenticated)" />`. The host page resolves `isAuthenticated` from `HttpContext.User.Identity.IsAuthenticated` (SSR page — `HttpContext` is available here). The `@rendermode InteractiveServer` directive is placed on the `<TournamentSetup>` component tag in the host page, or declared inside `TournamentSetup.razor` itself. Prefer declaring it inside the RCL component so the host page has no render mode knowledge.

For guest flow: the tournament state lives in `TournamentSetup` component fields. No DB call until login.
For organiser flow: `TournamentSetup` calls `ITournamentService.CreateAsync` on step 1 completion, then `AddParticipantAsync` on each add.

Gotcha: For the guest flow, the component must hold both participant names AND the created-tournament reference in memory. Separate the guest state from the organiser state using the `IsGuest` parameter to determine which code path to execute.

### Step 6.2 — Bracket Seeding Page

`src/TourneyGen.Components/Tournament/BracketSeeding.razor` (`@rendermode InteractiveServer`):

State:
- `List<string> _seededNames` — current seeding order (a reordered copy of participant names)
- `Guid _tournamentId` or `GuestTournamentState _guestState`

UI:
- Displays the seeded bracket pairs (who plays whom in round 1)
- "Reshuffle" button — calls `BracketEngine.Shuffle` in-memory and re-renders
- "Confirm and Start" button — for organiser: calls `ITournamentService.StartAsync`; for guest: persists seeding into `GuestTournamentState`
- Navigate to Bracket page on confirm

### Step 6.3 — Live Bracket Page

`src/TourneyGen.Components/Tournament/BracketView.razor` (`@rendermode InteractiveServer`):

State: `BracketViewModel _bracket` — refreshed after each result recording.

UI:
- Multi-column bracket graphic using CSS flexbox/grid
- One column per round, one card per match
- Each match card: Participant 1 name, Participant 2 name, Win button for each (disabled if already played)
- Completed match shows winner highlighted
- "Edit" button on completed matches — triggers cascade warning modal if needed

> [Blazor Review] **Implement `IAsyncDisposable` on `BracketView` and all other stateful InteractiveServer components.** The Blazor circuit (SignalR connection) has a configurable disconnect timeout (see Step 6.5). When a user closes the tab or navigates away, the circuit eventually disposes all components. If `BracketView` holds any subscriptions, timers, or `CancellationTokenSource` instances, they must be released in `DisposeAsync`. Even without explicit subscriptions, implement the interface as a forward-compatibility guard:
> ```csharp
> @implements IAsyncDisposable
> // ...
> @code {
>     private CancellationTokenSource _cts = new();
>
>     protected override async Task OnInitializedAsync()
>     {
>         _bracket = await TournamentService.GetBracketAsync(_tournamentId, _cts.Token);
>     }
>
>     public async ValueTask DisposeAsync()
>     {
>         await _cts.CancelAsync();
>         _cts.Dispose();
>     }
> }
> ```
> Pass `_cts.Token` to all service calls so they are cancelled when the circuit disconnects, preventing orphaned DB queries.

> [Blazor Review] **Loading data in `OnInitializedAsync` for InteractiveServer components.** `BracketView` loads `_bracket` from `ITournamentService.GetBracketAsync` in `OnInitializedAsync`. Because `@rendermode InteractiveServer` prerenders by default, this method runs once during SSR prerender and once when the interactive circuit connects. For a read-heavy operation like bracket loading, this double-execution is acceptable and produces the correct result (the data is the same both times). If prerendering should be disabled for performance reasons (e.g., the bracket load is expensive and the page shows nothing useful without interactivity), add `@rendermode @(new InteractiveServerRenderMode(prerender: false))` to the component. This is a deliberate trade-off: disabling prerender eliminates double-load but removes the instant server-rendered content on first load.

> [Blazor Review] **Accessing the authenticated user's ID inside `BracketView` (InteractiveServer).** Use the cascading `Task<AuthenticationState>` provided by `AddCascadingAuthenticationState()`:
> ```csharp
> [CascadingParameter]
> private Task<AuthenticationState> AuthenticationStateTask { get; set; } = default!;
>
> protected override async Task OnInitializedAsync()
> {
>     var authState = await AuthenticationStateTask;
>     _organiserId = authState.User.FindFirstValue(ClaimTypes.NameIdentifier);
>     _bracket = await TournamentService.GetBracketAsync(_tournamentId, _cts.Token);
> }
> ```
> Do NOT inject `IHttpContextAccessor` or use `[CascadingParameter] HttpContext` in this component — `HttpContext` is null once the SignalR circuit is active.

**Edit Result Flow:**
1. User clicks "Edit" on a completed match.
2. Component calls `ITournamentService.EditResultAsync(matchId, newWinnerId, confirmCascade: false)`.
3. If service throws `CascadeConfirmationRequiredException`: show `<ConfirmDialog>` with warning text listing affected downstream matches.
4. On confirm: call `EditResultAsync(matchId, newWinnerId, confirmCascade: true)`.
5. Refresh bracket.

**Participant Rename:**
- Inline edit icon next to each participant name wherever it appears in the bracket
- Opens an inline edit field — save calls `ITournamentService.RenameParticipantAsync`
- Changes are retroactive: all occurrences update because the bracket is re-fetched from `GetBracketAsync` which joins on participant name

> [Blazor Review] **Participant rename `EditForm` inside a list.** The rename inline edit is a small `EditForm` inside an `@foreach`. Each rename form needs its own ViewModel instance and its own `EditContext` — do not share a single model across all rename forms in the list. Use a dictionary keyed by `participantId` to hold per-participant edit state, or materialise the foreach with `@key="participant.Id"` to ensure Blazor correctly reconciles component instances:
> ```razor
> @foreach (var p in _bracket.AllParticipants)
> {
>     <div @key="p.Id">
>         <!-- rename edit form here, bound to a per-participant model -->
>     </div>
> }
> ```
> Always use `@key` with a stable identifier (the `Guid` participant ID) for lists of interactive sub-components. Without `@key`, Blazor may reuse component instances incorrectly when the list order changes after a reshuffle.

**Guest Bracket:**
- Same component, but all state operations mutate `GuestTournamentState` in memory instead of calling services
- Show "Log in to save your tournament" banner at the top

**Login prompt and migration:**
- When a guest clicks "Log in to save", show a modal linking to `/account/login?returnUrl=/tournament/guest-migrate`
- After login, `GuestMigrationService.MigrateToAccountAsync` is called (see Phase 3.8)

> [Blazor Review] **Guest migration and circuit-to-redirect state transfer.** The `GuestTournamentState` lives in the `BracketView` component's memory, which is scoped to the SignalR circuit. When the user clicks "Log in to save" and is redirected to `/account/login`, the **circuit is not transferred to the new page's HTTP request** — the Identity login flow is a standard HTTP redirect chain, not a circuit operation. This means the `GuestTournamentState` cannot be passed directly to the post-login callback via a DI scoped service.
>
> The correct migration approach:
> 1. Before triggering navigation to `/account/login`, **serialise the `GuestTournamentState` to `ProtectedLocalStorage`** (browser localStorage, ASP.NET Core data protection encrypted). This is the only cross-circuit persistence mechanism that does not require a database record.
> 2. After successful login (in the login page's `OnInitializedAsync` or a dedicated `/tournament/guest-migrate` SSR page), read the state back from `ProtectedLocalStorage` and call `GuestMigrationService.MigrateToAccountAsync`.
> 3. Clear the localStorage entry after successful migration.
>
> `ProtectedLocalStorage` is available as `IJSRuntime`-backed `ProtectedLocalStorage` — inject `ProtectedLocalStorage` from `Microsoft.AspNetCore.Components.Server.ProtectedBrowserStorage`. It requires the component to be `InteractiveServer` (calls must be in `OnAfterRenderAsync(firstRender)` since it uses JS interop). The write happens in `BracketView` before navigation; the read happens in a post-login component.
>
> Alternative simpler approach: Serialise `GuestTournamentState` to JSON and pass it as a query parameter on the login URL (only viable for small state — too large for typical tournament state). Use `ProtectedLocalStorage` as the primary approach.

### Step 6.4 — Delete Tournament UI

On the tournament detail pages: add a "Delete Tournament" button (red, at the bottom).
- Clicking opens `<ConfirmDialog>` with text "This will permanently delete the tournament and all its data. This cannot be undone."
- On confirm: call `ITournamentService.DeleteAsync(tournamentId)`, then navigate to `/dashboard`.

> [Blazor Review] After `DeleteAsync`, navigate with `Navigation.NavigateTo("/dashboard", forceLoad: false)`. The `forceLoad: false` triggers enhanced navigation (client-side navigation within the Blazor SPA context), which is faster than a full page reload. Only use `forceLoad: true` if the destination page has server-side auth state that must be refreshed from the cookie (e.g., after a sign-out).

### Step 6.5 — SignalR Circuit Connection Resilience

> [Blazor Review] **This step is required for all InteractiveServer pages** (bracket management, fixture entry, standings, leaderboard). A dropped SignalR connection mid-tournament leaves the user unable to record results until they reconnect or refresh. Configure the following:
>
> **1. Circuit disconnect timeout in `Program.cs`:**
> ```csharp
> builder.Services.AddRazorComponents()
>     .AddInteractiveServerComponents(options =>
>     {
>         options.DisconnectedCircuitMaxRetained = 100;
>         options.DisconnectedCircuitRetentionPeriod = TimeSpan.FromMinutes(3);
>     });
> ```
> The default `DisconnectedCircuitRetentionPeriod` is 3 minutes — this means a user who briefly loses network (e.g., mobile roaming) has 3 minutes to reconnect before the circuit (and any in-memory guest state) is discarded. For guest sessions with valuable in-progress state, this is the last line of defence before data loss. Do not increase this excessively — retained circuits consume server memory (approximately 250 KB each per architecture section 11.1).
>
> **2. Reconnect UI in `App.razor`:**
> Add the default Blazor reconnect UI components to `App.razor` so users see feedback when the connection drops:
> ```html
> <div id="blazor-error-ui">
>     An unhandled error has occurred.
>     <a href="" class="reload">Reload</a>
>     <a class="dismiss">X</a>
> </div>
> ```
> Style `#blazor-error-ui` in `app.css` to show/hide based on the Blazor-managed CSS classes `blazor-error-boundary`. The default template includes this — verify it is not accidentally removed.
>
> **3. Custom reconnect UI (recommended over the default):**
> For a better user experience on the bracket/fixture pages, add a visible reconnect overlay. Blazor fires `Blazor.defaultReconnectionHandler` events. In `App.razor`, add:
> ```html
> <div id="reconnect-modal" style="display:none;">
>     <p>Connection lost. Reconnecting...</p>
> </div>
> <script>
>     Blazor.start({
>         reconnectionOptions: {
>             maxRetries: 5,
>             retryIntervalMilliseconds: 3000
>         },
>         reconnectionHandler: {
>             onConnectionDown: () => document.getElementById('reconnect-modal').style.display = 'flex',
>             onConnectionUp: () => document.getElementById('reconnect-modal').style.display = 'none'
>         }
>     });
> </script>
> ```
> Place this before `</body>` and remove the auto-start default from `<script src="_framework/blazor.web.js"></script>` by adding `autostart="false"` to that script tag.
>
> **4. Nginx `proxy_read_timeout`:** Verify `docker/nginx/nginx.conf` has `proxy_read_timeout 3600s` (already specified in architecture section 13.2). This prevents Nginx from closing the WebSocket connection for long-running sessions. Without this, the default Nginx timeout (60s) kills the SignalR WebSocket and triggers a reconnect every minute.

---

## Phase 7 — Blazor UI: League Flow

**Depends on:** Phase 4 (LeagueService, FixtureGenerator, StandingsCalculator) and Phase 5.

### Step 7.1 — Create League Page

`src/TourneyGen.Components/League/LeagueSetup.razor` (`@rendermode InteractiveServer`):

Same pattern as `TournamentSetup.razor`. Key differences:
- Participant count: any integer 4–64 (number input, not dropdown)
- No seeding step — participants are added, league starts directly
- Route: `/league/create`, `@attribute [Authorize]` (leagues are account-only)

On "Start League": calls `ILeagueService.StartAsync(leagueId)` — this triggers `FixtureGenerator.GenerateFixtures`.

> [Blazor Review] **`[Authorize]` on RCL page components.** `@attribute [Authorize]` on a component inside `TourneyGen.Components` works correctly when `AuthorizeRouteView` is used in `Routes.razor` (Step 5.2). The `AuthorizeRouteView` inspects the `[Authorize]` attribute on the routed component and enforces it before rendering. However, the component is in the RCL — ensure the `Microsoft.AspNetCore.Authorization` namespace is in the RCL's `_Imports.razor` (already covered in Step 1.2 Blazor Review note). The `AuthorizeRouteView` redirect to the `<NotAuthorized>` template is triggered automatically; no explicit redirect code is needed inside the component.

> [Blazor Review] **Apply `IAsyncDisposable` to `LeagueSetup` and all InteractiveServer league components** (`FixtureList`, `StandingsTable`, `TiebreakerModal`) following the same pattern as `BracketView` (Step 6.3). Each component that calls services should hold a `CancellationTokenSource` and cancel it in `DisposeAsync`. This is especially important for `FixtureList` which may be loading large fixture sets (4,032 rows for 64 participants).

### Step 7.2 — Fixture Schedule Page

`src/TourneyGen.Components/League/FixtureList.razor` (`@rendermode InteractiveServer`):

State: `IReadOnlyList<FixtureGroupViewModel> _fixtureGroups` — fixtures grouped by round.

UI:
- Accordion or tabbed layout: one section per round (e.g., "Matchday 1", "Matchday 2")
- Each fixture row: Home participant, Away participant, outcome buttons (Home Win / Draw / Away Win)
- Played fixtures show the result with an "Edit" button
- Edit fixture: directly update (no cascade warning needed for leagues — league standings always recompute from all results)

> [Blazor Review] **Outcome recording does not need an `EditForm`** for the three-button (Home Win / Draw / Away Win) pattern — these are simple button click handlers that call `ILeagueService.RecordFixtureResultAsync` directly. Reserve `EditForm` + `DataAnnotationsValidator` for user text input forms (participant name entry). For single-selection UI like outcome buttons, use `@onclick` event handlers directly:
> ```razor
> <button class="btn btn-sm btn-success" @onclick="() => RecordResult(fixture.Id, MatchOutcome.HomeWin)">
>     Home Win
> </button>
> ```
> The "Edit" mode for a played fixture should show an inline row with the three outcome buttons (not a form), since there is no free-text input to validate.

> [Blazor Review] **`@key` on fixture rows is required.** Each fixture row must use `@key="fixture.Id"` (the `Guid` fixture ID). With 4,032 fixtures, Blazor's diffing algorithm needs stable keys to avoid re-rendering the entire list when one fixture's outcome changes. Without `@key`, updating a single fixture result triggers a full list reconciliation.

Gotcha: With 64 participants, there are 126 matchdays (rounds) and 4,032 fixtures. Render performance matters. Use virtualisation (`<Virtualize>` component) for the fixture list within each matchday group. Alternatively, paginate by round — show one matchday at a time with Previous/Next navigation. Pagination is simpler to implement correctly and avoids the height-estimation complexity of `<Virtualize>` with variable-height rows.

### Step 7.3 — Leaderboard Page

`src/TourneyGen.Components/League/StandingsTable.razor` (`@rendermode InteractiveServer`):

State: `IReadOnlyList<StandingRow> _standings` — refreshed after each fixture result.

UI:
- Ranked table: Rank, Name, P (played), W, D, L, Pts
- Shared ranks displayed as ties (e.g., two rows both showing "1")
- Highlighted top row(s) if league is complete
- If `AllFixturesPlayed && HasFirstPlaceTie && !TiebreakerResolved`: show "Tiebreaker Required" banner

> [Blazor Review] `StandingsTable` should implement `IAsyncDisposable` (same pattern as `BracketView`, Step 6.3). It loads standings data in `OnInitializedAsync` with a `CancellationToken`. After standings are loaded, the component's state is reactive: when a fixture result is recorded in `FixtureList`, `StandingsTable` must refresh its standings. The cleanest pattern for this sibling-component communication is a shared scoped service (a state container) injected into both components, or an `EventCallback` passed from a parent page component that hosts both `FixtureList` and `StandingsTable`. Do not use static events or singleton state containers — these leak between circuits.

**Tiebreaker Modal** (`src/TourneyGen.Components/League/TiebreakerModal.razor`):
- Shown when all fixtures are complete and first-place is tied
- Four options (radio buttons or cards):
  1. Auto-resolve (head-to-head)
  2. Generate tiebreaker bracket
  3. Manual selection (dropdown of tied participant names)
  4. Coin flip
- "Resolve" button — calls `ILeagueService.ResolveTiebreakerAsync` with selected method
- Shows result (winner name) on resolution, then "Complete League" button

> [Blazor Review] **`TiebreakerModal` uses `EditForm` for the manual selection and radio button options.** The tiebreaker selection UI has user-driven input (selecting a method, optionally selecting a participant from a dropdown). Wrap the selection in an `EditForm` with `DataAnnotationsValidator`:
> ```razor
> <EditForm Model="@_tiebreakerModel" OnValidSubmit="@HandleResolve">
>     <DataAnnotationsValidator />
>     <InputRadioGroup @bind-Value="_tiebreakerModel.Method">
>         <InputRadio Value="TiebreakerMethod.AutoHeadToHead" /> Auto (head-to-head)<br/>
>         <InputRadio Value="TiebreakerMethod.CoinFlip" /> Coin flip<br/>
>         <InputRadio Value="TiebreakerMethod.ManualSelection" /> Manual selection<br/>
>         <InputRadio Value="TiebreakerMethod.GenerateBracket" /> Generate bracket<br/>
>     </InputRadioGroup>
>     @if (_tiebreakerModel.Method == TiebreakerMethod.ManualSelection)
>     {
>         <InputSelect @bind-Value="_tiebreakerModel.ManualWinnerId">
>             @foreach (var p in _tiedParticipants)
>             {
>                 <option value="@p.Id">@p.DisplayName</option>
>             }
>         </InputSelect>
>         <ValidationMessage For="@(() => _tiebreakerModel.ManualWinnerId)" />
>     }
>     <button type="submit" class="btn btn-primary">Resolve</button>
> </EditForm>
> ```
> Server-side validation (via `DataAnnotationsValidator` in an `InteractiveServer` component) validates `ManualWinnerId` as required when `Method == ManualSelection` using a custom `IValidatableObject` or `[RequiredIf]`-style attribute.

### Step 7.4 — Delete League UI

Same pattern as tournament deletion. Navigate to `/dashboard` after deletion.

---

## Phase 8 — Sharing & Export

**Depends on:** Phase 4 (SharingService, ExportService) and Phases 6–7 (event pages).

### Step 8.1 — Add Share Button to Event Pages

On the tournament bracket page and league fixture/standings pages:
- "Share" button (account-only — `<AuthorizeView>`) — calls `ISharingService.GenerateLinkAsync`
- Displays the generated URL in a modal with a "Copy to clipboard" button
- Idempotent: clicking Share again shows the same URL

### Step 8.2 — Shared Tournament View

`src/TourneyGen.Web/Components/Pages/Share/SharedView.razor` — Static SSR, `@attribute [AllowAnonymous]`, route `/share/{token}`:

- Calls `ISharingService.ResolveTokenAsync(token)` during SSR render
- If null: render a generic "Not Found" page (no information about what was expected)
- If found and tournament: render read-only `BracketReadOnly` component (no action buttons, no edit, no win buttons)
- If found and league: render read-only `StandingsTableReadOnly` + `FixtureListReadOnly` components (no action buttons)
- Display "Last updated: {timestamp}" in the header

Key constraint: these views are Static SSR — no SignalR circuit. They are pure HTML renders. Create read-only variants of the bracket and fixture components that accept a `BracketViewModel` / `IReadOnlyList<StandingRow>` as parameters and render without any event callbacks.

> [Blazor Review] **Render mode correctness for shared views.** `SharedView.razor` is Static SSR (`@attribute [AllowAnonymous]`, no `@rendermode`). It must NOT render `BracketView.razor` or `FixtureList.razor` directly — those components declare `@rendermode InteractiveServer` internally, which would force a SignalR circuit onto the shared view page. This contradicts the architecture decision (architecture section 3.2) that shared views are static.
>
> The correct approach is to create dedicated read-only component variants with **no `@rendermode` directive**:
> - `src/TourneyGen.Components/Tournament/BracketReadOnly.razor` — Static SSR, accepts `BracketViewModel` as `[Parameter]`, renders the bracket grid with no buttons
> - `src/TourneyGen.Components/League/FixtureListReadOnly.razor` — Static SSR, accepts `IReadOnlyList<FixtureGroupViewModel>` as `[Parameter]`, renders fixture rows with outcome text only
> - `src/TourneyGen.Components/League/StandingsTableReadOnly.razor` — Static SSR, accepts `IReadOnlyList<StandingRow>` as `[Parameter]`, renders the standings table with no action controls
>
> These read-only components have no `[Inject]` service dependencies and no `@code` event handlers — they are pure render components. This makes them trivial to reuse in MAUI as well.
>
> **Do NOT use an `IsReadOnly` parameter approach on the existing interactive components** to toggle between interactive and read-only modes. While it avoids code duplication, it creates a risk: if `IsReadOnly=true` is passed but the component still declares `@rendermode InteractiveServer`, the circuit is still started for the shared view page, consuming server memory unnecessarily. Separate components are the correct separation of concerns.

> [Blazor Review] **Set HTTP 404 status for not-found tokens.** When `ResolveTokenAsync` returns null, the page must return HTTP 404 (not 200 with "not found" body content). SSR pages can set the response status code via the cascaded `HttpContext`:
> ```csharp
> [CascadingParameter]
> public HttpContext HttpContext { get; set; } = default!;
>
> protected override async Task OnInitializedAsync()
> {
>     _viewModel = await SharingService.ResolveTokenAsync(Token);
>     if (_viewModel is null)
>     {
>         HttpContext.Response.StatusCode = StatusCodes.Status404NotFound;
>     }
> }
> ```
> This is only possible on Static SSR pages — `HttpContext` is null once an interactive circuit takes over. The `SharedView.razor` page is correctly Static SSR, so this pattern applies.

### Step 8.3 — Export PDF Button

On archived event pages (dashboard archive section or event completion page):
- "Export PDF" button — for account holders only
- Calls `IExportService.GeneratePdfAsync(archivedEventId)` if `PdfStorageKey` is null
- Calls `IExportService.GetDownloadUrlAsync(storageKey)` to get a presigned URL
- Triggers browser download via JavaScript interop: `window.open(presignedUrl, '_blank')`

Gotcha: Presigned URL generation is a server-side operation. The URL is rendered into the page server-side, or the component calls the service and returns the URL to the client. Given the page uses Static SSR for archived events, the download is a POST form action that triggers PDF generation and returns a redirect to the presigned URL. Alternatively, use an InteractiveServer component just for the export button section.

> [Blazor Review] **JS interop for PDF download must use `OnAfterRenderAsync`.** The `window.open(presignedUrl, '_blank')` call requires `IJSRuntime` JS interop. JS interop is not available during SSR prerendering — it can only be called after the component has rendered in the browser (i.e., in `OnAfterRenderAsync(firstRender: true)` or from a button click handler). The recommended implementation:
>
> - Use an `InteractiveServer` component for the export button (`ExportPdfButton.razor` in `TourneyGen.Components`).
> - In the button's `@onclick` handler, call the services and then call JS interop:
>   ```csharp
>   private async Task HandleExportClick()
>   {
>       _isLoading = true;
>       var key = await ExportService.GeneratePdfAsync(_archivedEventId, _cts.Token);
>       var url = await ExportService.GetDownloadUrlAsync(key);
>       await JsRuntime.InvokeVoidAsync("open", url, "_blank");
>       _isLoading = false;
>   }
>   ```
> - The `IJSRuntime` injection in an `InteractiveServer` component is safe from a button click handler (the circuit is active). Do NOT call `IJSRuntime` from `OnInitializedAsync` — it is not safe during SSR prerendering.
> - This component is embedded in the (otherwise Static SSR) dashboard page by placing it on the page with `@rendermode InteractiveServer` on the component tag. This is the "island" pattern: a small interactive component on an otherwise static page.
>
> Do NOT use `IJSRuntime` in Static SSR pages for this feature. The alternative (pure SSR approach) is an HTML anchor tag with `download` attribute pointing to a server route (`/export/pdf/{archivedEventId}`) that generates the PDF and returns a `FileResult` with `Content-Disposition: attachment`. This avoids JS interop entirely and is simpler. Consider the anchor/route approach as the primary implementation for v1.

---

## Phase 9 — Polish & Constraints

**Depends on:** All previous phases functionally complete.

### Step 9.1 — Enforce 5-Event Cap Throughout UI

- Dashboard "Create New Event" button: query active event count. If count >= 5, show the button as disabled with a tooltip: "You've reached the maximum of 5 active events. Archive or delete an event to create a new one."
- Service layer throws `ActiveEventLimitException` as a safety net (this is the authoritative check).
- Handle the exception in Blazor components and show a user-friendly error message.

> [ASP.NET Review] **The service-layer check is the only reliable gate.** The UI button-disable is a UX convenience that can be bypassed by two concurrent browser tabs or a race condition. The `Serializable` transaction in `TournamentService.CreateAsync` and `LeagueService.CreateAsync` (Step 4.1) is the authoritative enforcement. Every code path that creates an event — direct dashboard create and guest migration via `GuestMigrationService.MigrateToAccountAsync` — must route through the same service method and the same transaction-protected cap check. There must be no bypass route.

### Step 9.2 — Delete Event Flow

Consistent delete flow across tournaments and leagues:
- "Delete" button visible on both active and archived events (on the event page and on dashboard event cards)
- Uses `<ConfirmDialog>` component with destructive action styling (red confirm button)
- Confirmation text: "Permanently delete [event name]? All data will be lost and cannot be recovered."
- After confirm: navigate to `/dashboard`
- Any active shared link for the event returns "not found" automatically (cascade delete in DB)

### Step 9.3 — 404 for Dead Shared Links

The `/share/{token}` route already handles this (Step 8.2). Add a dedicated `NotFound.razor` component in the Components library for reuse. Ensure the HTTP response status code is 404 (not 200) for not-found shared links — set it via `HttpContext.Response.StatusCode = 404` in the SSR page component via `[CascadingParameter] HttpContext HttpContext`.

### Step 9.4 — Responsive Layout

Apply Bootstrap 5 responsive grid to all pages:
- Use `container-lg` not `container-fluid` for readability on wide screens
- Bracket view: wrap in `overflow-x: auto` container. On mobile (< 768px), each round stacks or scrolls horizontally.
- Tables (standings, fixture list): add Bootstrap `table-responsive` wrapper
- Navigation: collapsible navbar (`navbar-expand-md`)

Test at 375px, 768px, and 1280px viewport widths.

### Step 9.5 — Error Handling and Logging

Configure Serilog in `Program.cs`:

```csharp
builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .WriteTo.Console()
    .WriteTo.File("logs/tourneygen-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 14));
```

Add `app.UseExceptionHandler("/error")` and create `Error.razor` (SSR) at route `/error` for unhandled exceptions.

Add structured logging at service method boundaries:
- `ILogger.LogInformation("Tournament {TournamentId} started by {OrganiserId}", ...)` on key operations
- `ILogger.LogWarning("Active event limit reached for {OrganiserId}", ...)` on cap violations
- `ILogger.LogError(ex, "Unexpected error in {Method}", ...)` on exceptions

> [ASP.NET Review] **Never log sensitive data.** Do not log connection strings, Gmail App Passwords, OAuth client secrets, S3 keys, or user email addresses at `Information` or higher levels. The templates above use `{TournamentId}` (a GUID) and `{OrganiserId}` (a GUID-based Identity string ID, not an email) — these are safe. The `Error.razor` page must never display exception stack traces or inner exception messages to end users — log them server-side and show a generic message in the browser.

### Step 9.6 — Input Validation

Add `DataAnnotations` validation to all Blazor forms (ViewModels/DTOs, not domain entities):
- Tournament/league name: `[Required, MaxLength(200)]`
- Participant name: `[Required, MaxLength(50)]`
- Participant count (tournament): `[Range(4, 64)]` + custom validation attribute for power-of-2
- Participant count (league): `[Range(4, 64)]`
- Display validation messages in forms via `<ValidationMessage>` components

> [Blazor Review] **Validation in `InteractiveServer` components runs server-side only.** This is a key distinction from Blazor WebAssembly. In `InteractiveServer`, `DataAnnotationsValidator` executes in the SignalR circuit on the server. There is no client-side JavaScript validation running in the browser. This means:
> - Validation messages appear after a round-trip to the server (typically < 100ms on a local network, sub-200ms over internet), which is fast enough for a real-time feel.
> - No custom JavaScript validation libraries (e.g., jQuery Validate, Parsley.js) are needed or appropriate.
> - `[Required]` and `[MaxLength]` work identically to how they work in Razor Pages server-side validation.
> - Do NOT add `[ValidateAntiForgeryToken]` attributes to form-handling methods inside `InteractiveServer` components — SignalR circuits are inherently session-bound and CSRF-safe. Antiforgery tokens are relevant only for SSR form posts (auth pages).
>
> **ViewModel placement:** ViewModel classes used by RCL components (e.g., `AddParticipantModel`, `TiebreakerSelectionModel`) should live in `TourneyGen.Application` (not `TourneyGen.Components`) so they can be referenced by both the component layer and any future API layer. They may use `System.ComponentModel.DataAnnotations` attributes — unlike domain entities (which must stay annotation-free), ViewModels in Application are presentation-layer objects and annotations are appropriate.
>
> **Server-side business rule validation beyond `DataAnnotations`:** After `OnValidSubmit` fires, the component calls the service. The service may throw business exceptions (e.g., `ActiveEventLimitException`, `TournamentLockedException`). Catch these in the component's `OnValidSubmit` handler and surface them as user-visible error messages. Do NOT let unhandled exceptions propagate — they will terminate the SignalR circuit and show the user the generic "An error has occurred. Reload the page." message:
> ```csharp
> private async Task HandleAddParticipant()
> {
>     try
>     {
>         await TournamentService.AddParticipantAsync(_tournamentId, _model.Name, _cts.Token);
>         _participants.Add(_model.Name);
>         _model = new(); // reset form
>     }
>     catch (TournamentLockedException)
>     {
>         _errorMessage = "The tournament has already started. No new participants can be added.";
>     }
>     catch (Exception ex)
>     {
>         Logger.LogError(ex, "Unexpected error adding participant to tournament {Id}", _tournamentId);
>         _errorMessage = "An unexpected error occurred. Please try again.";
>     }
> }
> ```

---

## Phase 10 — Testing & Deployment

**Depends on:** All application code complete from previous phases.

### Step 10.1 — Unit Tests: BracketEngine

In `tests/TourneyGen.Application.Tests/Tournaments/`:

- `BracketEngine_Shuffle_ProducesPermutation` — verify output is a permutation of input
- `BracketEngine_Shuffle_IsNotSorted` — verify statistical randomness (run N times, check distribution)
- `BracketEngine_BuildBracketViewModel_CorrectRoundCount` — for n=4: 2 rounds, for n=64: 6 rounds
- `BracketEngine_BuildBracketViewModel_CorrectMatchCount` — round 1 has n/2 matches, round 2 has n/4, etc.
- `BracketEngine_BuildBracketViewModel_WinnerPropagated` — after recording round 1 match, winner appears in round 2 slot

### Step 10.2 — Unit Tests: FixtureGenerator

In `tests/TourneyGen.Application.Tests/Leagues/`:

- `FixtureGenerator_EvenParticipants_CorrectFixtureCount` — for n participants, expect n*(n-1) fixtures
- `FixtureGenerator_OddParticipants_CorrectFixtureCount` — odd n handled correctly (no BYE fixtures in output)
- `FixtureGenerator_EachPairPlaysExactlyTwice` — verify each home/away pair appears exactly once in each leg
- `FixtureGenerator_NoParticipantPlaysItself` — verify HomeId != AwayId for all fixtures

### Step 10.3 — Unit Tests: StandingsCalculator

In `tests/TourneyGen.Application.Tests/Leagues/`:

- `StandingsCalculator_WinIsThreePoints`
- `StandingsCalculator_DrawIsOnePointEach`
- `StandingsCalculator_LossIsZeroPoints`
- `StandingsCalculator_TiedParticipantsShareRank`
- `StandingsCalculator_RankGapAfterTie` — if ranks 1 and 2 tie at position 1, next rank is 3
- `StandingsCalculator_CorrectGamesPlayedCount`
- `StandingsCalculator_UnplayedFixturesNotCounted`

### Step 10.4 — Unit Tests: Service Layer (Mocked)

In `tests/TourneyGen.Application.Tests/`:

Mock `AppDbContext` via an in-memory EF Core provider (or use `Moq` for the DbSet).

- `TournamentService_CreateAsync_ThrowsWhenAtCap` — mock 5 existing active events, verify `ActiveEventLimitException`
- `TournamentService_CreateAsync_InvalidParticipantCount_Throws` — test n=5 (not power of 2)
- `TournamentService_AddParticipant_ThrowsWhenLocked`
- `TournamentService_RecordResult_SetsWinnerAndLoser`
- `TournamentService_RecordResult_FinalMatch_CompletesTournament`
- `TournamentService_EditResult_ThrowsCascadeConfirmation_WhenDownstreamResultsExist`
- `LeagueService_StartAsync_GeneratesFixtures`
- `LeagueService_GetStandingsAsync_ReturnsCorrectRanks`

### Step 10.5 — Integration Tests

In `tests/TourneyGen.Integration.Tests/`:

Use `Microsoft.EntityFrameworkCore.InMemory` provider (or Testcontainers with MySQL for full fidelity).

- Full tournament flow: create → add participants → start → record all results → verify complete
- Full league flow: create → add participants → start → record all fixtures → get standings
- Sharing: create tournament → generate link → resolve token → verify content
- Active event cap: create 5 events → attempt 6th → verify rejection

Gotcha: Integration tests with MySQL (via Testcontainers) are more reliable than in-memory because Pomelo-specific behaviours (check constraints, cascade rules) are not tested with the in-memory provider. For v1, in-memory is acceptable for speed; add Testcontainers before production.

> [ASP.NET Review] **Testcontainers MySQL setup.** Add `Testcontainers.MySql` (4.x) to `TourneyGen.Integration.Tests`. Create a `MySqlContainerFixture` implementing xUnit's `IAsyncLifetime` that starts `mysql:8.0` once per test collection. Each test wraps its operations in a transaction rolled back in `DisposeAsync`. This gives real MySQL semantics (check constraints, `SERIALIZABLE` isolation for the cap test) without requiring a running MySQL instance in CI — Testcontainers pulls the image from Docker Hub automatically. Also prefer EF Core InMemory provider over raw Moq for `DbSet<T>` in unit tests — it is far less boilerplate and supports real LINQ semantics.

### Step 10.6 — Docker Compose Production Configuration

Review `docker/docker-compose.yml` from Phase 1 and verify:
- `ASPNETCORE_ENVIRONMENT=Production` for the app service
- No exposed ports on `db` service (only accessible via internal Docker network)
- `nginx` service maps ports 80 and 443 to host
- Health check on `db` before `app` starts
- `restart: unless-stopped` on all services
- Volumes named and persisted for `mysql_data`

Create `docker/docker-compose.override.yml` for development — override to expose MySQL port 3306 and disable Nginx (direct HTTP on port 5000).

### Step 10.7 — Production Nginx Configuration

Finalize `docker/nginx/nginx.conf`:
- Replace `tourneygen.example.com` with the real domain
- Configure SSL with Let's Encrypt paths
- Ensure `proxy_read_timeout 3600s` for SignalR circuits
- Include `Upgrade` and `Connection` headers for WebSocket

Set up Certbot on the VPS host (not inside Docker) to manage certificates at `/etc/letsencrypt`. Mount the certificate directory into the Nginx container as a read-only volume.

### Step 10.8 — Deploy to Linode VPS

Pre-deployment checklist:
1. Provision Linode VPS (recommended: 4 GB RAM, 2 vCPU Nanode or Dedicated plan).
2. Install Docker and Docker Compose on the VPS.
3. Create `.env` on the VPS with production credentials.
4. Create Linode Object Storage bucket for PDFs. Create separate bucket for DB backups.
5. Obtain Linode Object Storage S3 access keys.
6. Create Google OAuth app, set production redirect URI.
7. Enable Gmail SMTP with App Password.
8. Copy `docker/` directory to VPS.
9. Run `docker compose pull && docker compose up -d`.

Migration run (production):
```bash
docker compose run --rm app dotnet ef database update \
  --project src/TourneyGen.Infrastructure \
  --startup-project src/TourneyGen.Web
```

Or build and run a migration runner container as a pre-deployment step.

> [ASP.NET Review] **`dotnet ef` is NOT available in the production runtime image.** The Dockerfile uses `mcr.microsoft.com/dotnet/aspnet:8.0` as the final runtime image, which does NOT include the `dotnet ef` CLI. Running `docker compose run --rm app dotnet ef database update` against the production image fails with "No executable found matching command dotnet-ef". Use instead: add a startup migration path in `Program.cs` controlled by an environment variable:
> ```csharp
> if (Environment.GetEnvironmentVariable("RUN_MIGRATIONS") == "true")
> {
>     using var scope = app.Services.CreateScope();
>     var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
>     await db.Database.MigrateAsync();
>     return; // Exit after migration
> }
> ```
> Run before the normal `docker compose up -d`:
> ```bash
> docker compose run --rm -e RUN_MIGRATIONS=true app
> docker compose up -d
> ```

### Step 10.9 — Backup Configuration

On the VPS host (outside Docker), create `/etc/cron.daily/tourneygen-backup` per architecture section 13.4. Make it executable. Test with a manual run. Verify the backup file appears in the Linode Object Storage backup bucket.

### Step 10.10 — Smoke Test Production Deployment

Manual checklist:
- [ ] Landing page loads at `https://yourdomain.com`
- [ ] Registration email is received and confirms successfully
- [ ] Google OAuth sign-in creates account
- [ ] Create tournament as guest (end-to-end: add participants, shuffle seeding, record all results, bracket completes)
- [ ] Guest login migrates tournament to account
- [ ] Create league as organiser (add participants, start, record fixtures, view standings)
- [ ] Tiebreaker resolves correctly (test at least coin flip)
- [ ] Shared link generates and resolves read-only view
- [ ] Deleted event causes shared link to return 404
- [ ] PDF export generates and downloads
- [ ] 5-event cap blocks a 6th creation
- [ ] Delete event removes all data

---

## Dependency Summary

```
Phase 1 (Scaffold)
  └─> Phase 2 (Domain & Data Model)
        └─> Phase 3 (Authentication)
              └─> Phase 4 (Application Services)
                    └─> Phase 5 (Core Pages)
                          ├─> Phase 6 (Tournament UI)
                          ├─> Phase 7 (League UI)
                          └─> Phase 8 (Sharing & Export)
                                └─> Phase 9 (Polish)
                                      └─> Phase 10 (Testing & Deployment)
```

Phases 6 and 7 are largely parallel once Phase 5 is complete. Phase 8 depends on both 6 and 7 for the share buttons being in place. Testing (Phase 10) can be started in parallel with Phase 5+ — unit tests for domain logic can be written as soon as Phase 2 is complete.

---

### Critical Files for Implementation

- `/home/lrdtms/Project/TourneyGen/src/TourneyGen.Infrastructure/Data/AppDbContext.cs`
- `/home/lrdtms/Project/TourneyGen/src/TourneyGen.Web/Program.cs`
- `/home/lrdtms/Project/TourneyGen/src/TourneyGen.Application/Tournaments/ITournamentService.cs`
- `/home/lrdtms/Project/TourneyGen/src/TourneyGen.Application/Leagues/FixtureGenerator.cs`
- `/home/lrdtms/Project/TourneyGen/src/TourneyGen.Infrastructure/DependencyInjection.cs`
