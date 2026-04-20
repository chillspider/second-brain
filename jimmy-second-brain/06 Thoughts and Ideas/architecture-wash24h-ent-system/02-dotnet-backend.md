# 02 — .NET 9 Backend

> Clean Architecture solution, ASP.NET Core minimal APIs, EF Core → PostgreSQL, Keycloak JWT validation.

---

## 1. Solution scaffolding

```bash
cd backend
dotnet new sln -n JetX

# Layers (inner to outer)
dotnet new classlib -n JetX.Domain         -o src/JetX.Domain           -f net9.0
dotnet new classlib -n JetX.Application    -o src/JetX.Application      -f net9.0
dotnet new classlib -n JetX.Infrastructure -o src/JetX.Infrastructure   -f net9.0
dotnet new webapi   -n JetX.Api            -o src/JetX.Api              -f net9.0 --use-minimal-apis

# Tests
dotnet new xunit -n JetX.Domain.Tests      -o tests/JetX.Domain.Tests
dotnet new xunit -n JetX.Application.Tests -o tests/JetX.Application.Tests
dotnet new xunit -n JetX.Api.Tests         -o tests/JetX.Api.Tests

# Wire references
dotnet sln add $(find . -name "*.csproj")
dotnet add src/JetX.Application      reference src/JetX.Domain
dotnet add src/JetX.Infrastructure   reference src/JetX.Application
dotnet add src/JetX.Api              reference src/JetX.Application src/JetX.Infrastructure
```

### Dependency direction (enforce, don't just document)

```
Api ──▶ Application ──▶ Domain
  └──▶ Infrastructure ──▶ Application ──▶ Domain
```

`Domain` depends on **nothing**. No EF Core, no ASP.NET, no NuGet packages beyond `System.*`.

Consider adding `NetArchTest.Rules` in a test to fail the build if someone adds a forbidden reference.

---

## 2. Packages

### JetX.Api

```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="9.0.*" />
<PackageReference Include="Keycloak.AuthServices.Authentication" Version="2.9.*" />
<PackageReference Include="Keycloak.AuthServices.Authorization" Version="2.9.*" />
<PackageReference Include="Serilog.AspNetCore" Version="9.0.*" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.10.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.10.*" />
<PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.10.0-*" />
<PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.0.*" />
<PackageReference Include="Scalar.AspNetCore" Version="2.*" />  <!-- OpenAPI UI -->
```

### JetX.Application

```xml
<PackageReference Include="MediatR" Version="12.*" />
<PackageReference Include="FluentValidation" Version="11.*" />
<PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.*" />
```

### JetX.Infrastructure

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.0.*" />
```

`Keycloak.AuthServices` is the canonical community library for Keycloak ↔ ASP.NET Core integration. It knows about realm roles, client roles, and Keycloak Organizations (v24+).

---

## 3. Configuration

**`appsettings.json`:**

```json
{
  "Keycloak": {
    "realm": "jetx",
    "auth-server-url": "https://auth.wash24h.com/",
    "ssl-required": "external",
    "resource": "jetx-api",
    "verify-token-audience": true,
    "credentials": {
      "secret": "set-via-user-secrets-or-env"
    },
    "confidential-port": 0
  },
  "ConnectionStrings": {
    "AppDb": "Host=hcm-db-a;Database=jetx_app;Username=jetx_app;Password=set-via-secrets"
  },
  "Cors": {
    "AllowedOrigins": ["https://app.wash24h.com"]
  }
}
```

**Dev secrets:**

```bash
cd src/JetX.Api
dotnet user-secrets init
dotnet user-secrets set "Keycloak:credentials:secret" "dev-secret-from-keycloak"
dotnet user-secrets set "ConnectionStrings:AppDb" "Host=localhost;Database=jetx_app_dev;Username=jetx;Password=dev"
```

**Production:** read from `/etc/jetx/api.env` loaded by systemd `EnvironmentFile=`. Variables with `__` replace `:` in config paths:

```
Keycloak__credentials__secret=<real-secret>
ConnectionStrings__AppDb=Host=hcm-db-a;Database=jetx_app;Username=jetx_app;Password=<real>
```

---

## 4. `Program.cs` — the wiring

```csharp
using Keycloak.AuthServices.Authentication;
using Keycloak.AuthServices.Authorization;
using Microsoft.EntityFrameworkCore;
using JetX.Infrastructure;
using JetX.Application;

var builder = WebApplication.CreateBuilder(args);

// ─── Logging (Serilog to stdout + JSON for container collectors) ──────────
builder.Host.UseSerilog((ctx, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter()));

// ─── Auth: validate JWTs issued by Keycloak ───────────────────────────────
builder.Services
    .AddKeycloakWebApiAuthentication(builder.Configuration)
    .AddJwtBearer();

builder.Services
    .AddAuthorization()
    .AddKeycloakAuthorization(options =>
    {
        // Map both realm_access.roles and resource_access.<client>.roles into claims
        options.EnableRolesMapping =
            Keycloak.AuthServices.Authorization.RolesClaimTransformationSource.All;
        options.RolesResource = "jetx-api"; // where client roles live
    })
    .AddAuthorizationBuilder()
        // Coarse policies used by endpoints
        .AddPolicy("ops",          p => p.RequireRealmRoles("ops"))
        .AddPolicy("admin",        p => p.RequireRealmRoles("platform-admin"))
        .AddPolicy("sites:read",   p => p.RequireResourceRoles("sites:read"))
        .AddPolicy("sites:write",  p => p.RequireResourceRoles("sites:write"))
        .AddPolicy("cameras:read", p => p.RequireResourceRoles("cameras:read"))
        .AddPolicy("cameras:ctrl", p => p.RequireResourceRoles("cameras:control"));

// ─── EF Core ──────────────────────────────────────────────────────────────
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("AppDb"))
       .UseSnakeCaseNamingConvention());

// ─── Application layer (MediatR + validators) ─────────────────────────────
builder.Services.AddApplication();       // extension method in JetX.Application
builder.Services.AddInfrastructure();    // extension method in JetX.Infrastructure

// ─── CORS ─────────────────────────────────────────────────────────────────
var allowedOrigins = builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>()!;
builder.Services.AddCors(o => o.AddDefaultPolicy(p =>
    p.WithOrigins(allowedOrigins).AllowAnyHeader().AllowAnyMethod().AllowCredentials()));

// ─── OpenAPI + Scalar UI ──────────────────────────────────────────────────
builder.Services.AddOpenApi();

// ─── Observability ────────────────────────────────────────────────────────
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddPrometheusExporter());

// ─── Rate limiting (public endpoints only) ────────────────────────────────
builder.Services.AddRateLimiter(opt =>
{
    opt.AddFixedWindowLimiter("anon", o =>
    {
        o.Window = TimeSpan.FromMinutes(1);
        o.PermitLimit = 60;
    });
});

var app = builder.Build();

app.UseSerilogRequestLogging();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapOpenApi();
app.MapScalarApiReference();          // https://api.wash24h.com/scalar
app.MapPrometheusScrapingEndpoint();  // /metrics

// Endpoint modules (one file per bounded context)
app.MapSitesEndpoints();
app.MapCamerasEndpoints();
app.MapUsersEndpoints();

app.Run();
```

---

## 5. Endpoint pattern

One static class per feature, registered in `Program.cs`. Keeps `Program.cs` flat.

**`JetX.Api/Features/Sites/SitesEndpoints.cs`:**

```csharp
public static class SitesEndpoints
{
    public static IEndpointRouteBuilder MapSitesEndpoints(this IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/sites")
                   .RequireAuthorization()      // all endpoints require login
                   .WithTags("Sites");

        g.MapGet("/", ListSites).RequireAuthorization("sites:read");
        g.MapGet("/{id:guid}", GetSite).RequireAuthorization("sites:read");
        g.MapPost("/", CreateSite).RequireAuthorization("sites:write");
        g.MapPut("/{id:guid}", UpdateSite).RequireAuthorization("sites:write");

        return app;
    }

    static async Task<Ok<IReadOnlyList<SiteDto>>> ListSites(
        IMediator mediator, CancellationToken ct) =>
            TypedResults.Ok(await mediator.Send(new ListSitesQuery(), ct));

    static async Task<Results<Ok<SiteDto>, NotFound>> GetSite(
        Guid id, IMediator mediator, CancellationToken ct)
    {
        var site = await mediator.Send(new GetSiteQuery(id), ct);
        return site is null ? TypedResults.NotFound() : TypedResults.Ok(site);
    }

    // ... CreateSite, UpdateSite
}
```

---

## 6. Domain → Application → Infrastructure

### Domain (pure C#)

**`JetX.Domain/Entities/Site.cs`:**

```csharp
namespace JetX.Domain.Entities;

public sealed class Site
{
    public Guid Id { get; private set; }
    public string Code { get; private set; }          // e.g. "HCM-PLX-30"
    public string Name { get; private set; }
    public Province Province { get; private set; }
    public StationTier Tier { get; private set; }
    public bool IsActive { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private Site() { /* EF */ }

    public static Site Create(string code, string name, Province province, StationTier tier)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(code);
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        return new Site
        {
            Id = Guid.CreateVersion7(),
            Code = code.ToUpperInvariant(),
            Name = name,
            Province = province,
            Tier = tier,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void Deactivate() => IsActive = false;
}

public enum StationTier { Premium, Standard, Challenger }
```

### Application (CQRS with MediatR)

**`JetX.Application/Sites/Queries/ListSitesQuery.cs`:**

```csharp
public record ListSitesQuery() : IRequest<IReadOnlyList<SiteDto>>;

public record SiteDto(Guid Id, string Code, string Name, string Province, string Tier, bool IsActive);

internal sealed class ListSitesHandler(IAppDbContext db)
    : IRequestHandler<ListSitesQuery, IReadOnlyList<SiteDto>>
{
    public async Task<IReadOnlyList<SiteDto>> Handle(ListSitesQuery req, CancellationToken ct) =>
        await db.Sites
            .AsNoTracking()
            .OrderBy(s => s.Code)
            .Select(s => new SiteDto(s.Id, s.Code, s.Name, s.Province.ToString(), s.Tier.ToString(), s.IsActive))
            .ToListAsync(ct);
}
```

### Infrastructure (EF Core)

**`JetX.Infrastructure/Persistence/AppDbContext.cs`:**

```csharp
public interface IAppDbContext
{
    DbSet<Site> Sites { get; }
    Task<int> SaveChangesAsync(CancellationToken ct);
}

public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options), IAppDbContext
{
    public DbSet<Site> Sites => Set<Site>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

## 7. User context from Keycloak token

Inside handlers that need "who is calling":

```csharp
public interface ICurrentUser
{
    Guid KeycloakId { get; }           // sub claim
    string Username { get; }
    IReadOnlyList<string> Roles { get; }
    bool IsInGroup(string group);
}

internal sealed class CurrentUser(IHttpContextAccessor http) : ICurrentUser
{
    ClaimsPrincipal? U => http.HttpContext?.User;

    public Guid KeycloakId => Guid.Parse(U!.FindFirstValue(ClaimTypes.NameIdentifier)!);
    public string Username => U!.FindFirstValue("preferred_username") ?? "";
    public IReadOnlyList<string> Roles =>
        U!.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList();
    public bool IsInGroup(string g) =>
        U!.FindAll("groups").Any(c => c.Value == g);
}
```

Register: `services.AddHttpContextAccessor(); services.AddScoped<ICurrentUser, CurrentUser>();`

---

## 8. Migrations

```bash
cd backend
dotnet ef migrations add Initial \
    --project src/JetX.Infrastructure \
    --startup-project src/JetX.Api \
    --output-dir Persistence/Migrations

dotnet ef database update \
    --project src/JetX.Infrastructure \
    --startup-project src/JetX.Api
```

In production, run `dotnet ef migrations bundle` to produce a self-contained migrator binary; invoke it from the systemd `ExecStartPre=` of the API service so migrations apply before the API starts.

---

## 9. Health and readiness

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("db")
    .AddUrlGroup(new Uri("https://auth.wash24h.com/realms/jetx/.well-known/openid-configuration"),
                 name: "keycloak", failureStatus: HealthStatus.Degraded);

app.MapHealthChecks("/health/live",
    new HealthCheckOptions { Predicate = _ => false }); // process alive only
app.MapHealthChecks("/health/ready"); // all checks
```

---

## 10. Testing

- **Domain tests:** pure logic, no mocks needed
- **Application tests:** use `Testcontainers.PostgreSql` to spin up a real PostgreSQL per test class, test handlers against it
- **Api tests:** `WebApplicationFactory<Program>` with a fake auth scheme that injects claims, no real Keycloak

**Fake auth for tests:**

```csharp
public class TestAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> o,
    ILoggerFactory l, UrlEncoder u) : AuthenticationHandler<AuthenticationSchemeOptions>(o, l, u)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[] {
            new Claim(ClaimTypes.NameIdentifier, "test-user-id"),
            new Claim("preferred_username", "test"),
            new Claim(ClaimTypes.Role, "ops"),
            new Claim(ClaimTypes.Role, "sites:read"),
            new Claim(ClaimTypes.Role, "sites:write"),
        };
        var identity = new ClaimsIdentity(claims, "Test");
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), "Test");
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

---

## 11. Things to avoid

- **Don't store users in your own DB keyed by email/username.** Key domain tables by `keycloak_sub` (Guid). If you need a local user projection for joins, sync it from Keycloak events, don't duplicate auth state.
- **Don't call Keycloak admin API on every request.** Cache realm public keys (JWT library does this automatically), cache user lookups with a short TTL.
- **Don't put business rules in controllers/endpoints.** They belong in handlers. Endpoints only orchestrate: validate shape, call MediatR, shape response.
- **Don't use `async void` anywhere.** Ever.
- **Don't swallow exceptions.** Let them bubble. A single exception-handling middleware converts them to ProblemDetails.
