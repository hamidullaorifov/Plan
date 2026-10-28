# 9. Integration Testing: WebApplicationFactory, Testcontainers and Respawn

> Phase 2 · Week 12 · [← Phase index](README.md)

**Why this matters:** Unit tests can't catch a wrong SQL index, a broken migration, a missing middleware, a misconfigured auth policy or a race condition. **Integration tests against real infrastructure** catch exactly the bugs that reach production. Testcontainers makes them reliable and runnable on any machine and in CI.

---

## 9.1 The setup

```
tests/TicketHub.Api.IntegrationTests
 ├─ ApiFactory.cs              WebApplicationFactory<Program> + Postgres/Redis containers
 ├─ TestAuthHandler.cs         a fake authentication scheme
 ├─ ApiCollection.cs           shares one factory (and its containers) across test classes
 └─ Holds/HoldSeatsTests.cs    the tests
```

```powershell
dotnet new xunit3 -n TicketHub.Api.IntegrationTests -o tests/TicketHub.Api.IntegrationTests -f net10.0
dotnet add tests/TicketHub.Api.IntegrationTests reference src/TicketHub.Api
dotnet add tests/TicketHub.Api.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing
dotnet add tests/TicketHub.Api.IntegrationTests package Testcontainers.PostgreSql
dotnet add tests/TicketHub.Api.IntegrationTests package Testcontainers.Redis
dotnet add tests/TicketHub.Api.IntegrationTests package Respawn
dotnet add tests/TicketHub.Api.IntegrationTests package Shouldly
```

> Since .NET 10, a source generator makes the top-level `Program` class public, so `WebApplicationFactory<Program>` works without adding `public partial class Program {}` yourself.

---

## 9.2 The factory

```csharp
public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder("postgres:17-alpine").Build();
    private readonly RedisContainer _redis = new RedisBuilder("redis:7-alpine").Build();
    private Respawner _respawner = null!;

    public async ValueTask InitializeAsync()
    {
        await Task.WhenAll(_postgres.StartAsync(), _redis.StartAsync());

        using (var scope = Services.CreateScope())                 // building Services starts the host
        {
            var db = scope.ServiceProvider.GetRequiredService<TicketHubDbContext>();
            await db.Database.MigrateAsync();                      // tests the real migrations
        }

        await using var connection = new NpgsqlConnection(_postgres.GetConnectionString());
        await connection.OpenAsync();
        _respawner = await Respawner.CreateAsync(connection, new RespawnerOptions
        {
            DbAdapter = DbAdapter.Postgres,
            SchemasToInclude = ["public"],
            TablesToIgnore = ["__EFMigrationsHistory"],
        });
    }

    public async Task ResetDatabaseAsync()
    {
        await using var connection = new NpgsqlConnection(_postgres.GetConnectionString());
        await connection.OpenAsync();
        await _respawner.ResetAsync(connection);
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.UseSetting("ConnectionStrings:tickethub", _postgres.GetConnectionString());
        builder.UseSetting("ConnectionStrings:redis", _redis.GetConnectionString());

        builder.ConfigureTestServices(services =>
        {
            services.AddAuthentication(TestAuthHandler.Scheme)
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(TestAuthHandler.Scheme, _ => { });
            services.PostConfigure<AuthenticationOptions>(o =>
            {
                o.DefaultAuthenticateScheme = TestAuthHandler.Scheme;
                o.DefaultChallengeScheme = TestAuthHandler.Scheme;
            });
        });
    }

    public HttpClient CreateClientFor(Guid userId, params string[] roles)
    {
        var client = CreateClient();
        client.DefaultRequestHeaders.Add(TestAuthHandler.UserHeader, userId.ToString());
        client.DefaultRequestHeaders.Add(TestAuthHandler.RolesHeader, string.Join(',', roles));
        return client;
    }

    public override async ValueTask DisposeAsync()
    {
        await base.DisposeAsync();
        await _postgres.DisposeAsync();
        await _redis.DisposeAsync();
    }
}
```

> Testcontainers APIs change between major versions. If a builder constructor or method shows an obsolete warning, follow the hint and check the Testcontainers for .NET docs for the current pattern.

---

## 9.3 A fake authentication scheme

```csharp
public sealed class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string Scheme = "Test";
    public const string UserHeader = "X-Test-User";
    public const string RolesHeader = "X-Test-Roles";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(UserHeader, out var userId))
            return Task.FromResult(AuthenticateResult.NoResult());               // anonymous → policies return 401

        var claims = new List<Claim> { new("sub", userId.ToString()) };
        foreach (var role in Request.Headers[RolesHeader].ToString().Split(',', StringSplitOptions.RemoveEmptyEntries))
            claims.Add(new Claim("roles", role));

        var identity = new ClaimsIdentity(claims, Scheme, nameType: "sub", roleType: "roles");
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme)));
    }
}
```

This tests **your authorization rules** (policies, ownership) without depending on Keycloak. Keep a small number of separate end-to-end tests against real Keycloak if you need them.

---

## 9.4 Sharing containers across test classes

```csharp
[CollectionDefinition(Name)]
public sealed class ApiCollection : ICollectionFixture<ApiFactory>
{
    public const string Name = "api";
}

[Collection(ApiCollection.Name)]
public sealed class HoldSeatsTests(ApiFactory factory) : IAsyncLifetime
{
    public async ValueTask InitializeAsync() => await factory.ResetDatabaseAsync();   // a clean database per test
    public ValueTask DisposeAsync() => ValueTask.CompletedTask;

    // tests...
}
```

Starting containers takes seconds, so **start them once per collection** and **reset the data per test** with Respawn, which deletes rows quickly and in the right order.

---

## 9.5 The test that matters: race-safe seat holds

```csharp
[Fact]
public async Task Only_one_of_many_concurrent_holds_for_the_same_seat_succeeds()
{
    var ct = TestContext.Current.CancellationToken;
    var eventId = await TestData.CreatePublishedEventAsync(factory, seats: ["A-1"], ct);

    var attempts = Enumerable.Range(0, 20).Select(async _ =>
    {
        using var client = factory.CreateClientFor(Guid.NewGuid(), "customer");
        using var response = await client.PostAsJsonAsync(
            $"/api/events/{eventId}/holds", new { seatCodes = new[] { "A-1" } }, ct);
        return response.StatusCode;
    });

    var statuses = await Task.WhenAll(attempts);

    statuses.Count(s => s == HttpStatusCode.Created).ShouldBe(1);
    statuses.Count(s => s == HttpStatusCode.Conflict).ShouldBe(19);
}
```

> Run it many times (for example in a loop of 50). A concurrency test that passes once proves little. If it **ever** fails, you have a real race condition.

### Other must-have integration tests

| Test | Proves |
|---|---|
| Anonymous `POST /api/events/{id}/holds` → 401 | auth is wired up and the fallback policy works |
| A customer creating an event → 403 | role policies work |
| User B reading user A's order → 404 | there's no IDOR |
| An invalid request body → 400 ProblemDetails with field errors | validation and the error contract |
| Holding an expired hold's seats succeeds | the expiry logic works against real SQL and times |
| Checking out twice with the same `Idempotency-Key` → one order, the same body | idempotency |
| `GET /api/events` is served from the output cache the second time | the caching wiring |

---

## 9.6 Running in CI

GitHub-hosted Ubuntu runners have Docker, so **Testcontainers works out of the box**:

```yaml
- run: dotnet test --no-build -c Release
```

**Tips:**

- Pin image tags (`postgres:17-alpine`) so the tests stay reproducible.
- Keep integration tests in a separate project, so unit tests stay fast locally (`dotnet test tests/TicketHub.Domain.Tests`).
- Use the xUnit v3 `TestContext.Current.CancellationToken` so cancelled CI runs stop quickly.

---

## Interview questions

<details><summary><b>1. Why use Testcontainers instead of EF InMemory or SQLite for tests?</b></summary>

InMemory and SQLite behave differently from production PostgreSQL or SQL Server: transactions, constraints, concurrency tokens, SQL translation, collation and time handling all differ. Bugs hide in exactly those differences. Testcontainers runs the real engine in Docker.
</details>

<details><summary><b>2. How do you test authorization without a real identity provider?</b></summary>

Replace the authentication scheme with a test handler that builds a `ClaimsPrincipal` from headers. Policies and handlers still run, so you test your rules, including the negative cases.
</details>

<details><summary><b>3. How do you keep integration tests fast and isolated?</b></summary>

Share containers per collection or assembly with fixtures, reset the data per test (Respawn or transaction rollback), and avoid sleeps by using `FakeTimeProvider` where time matters. Run independent collections in parallel.
</details>

<details><summary><b>4. How would you test a race condition?</b></summary>

Fire many concurrent requests at the same resource (with `Task.WhenAll`), assert the invariant (exactly one success), and repeat the test many times. Use real databases, because in-memory fakes don't reproduce locking or concurrency tokens.
</details>

---

## Exercises

1. Build `ApiFactory`, `TestAuthHandler` and the collection fixture, and get one smoke test green in CI.
2. Write every test in the "must-have" table.
3. Temporarily remove the concurrency token from `EventSeat` and watch the race test fail. Put it back. Write down what you observed.
4. Time the suite. If it's slower than about 60 seconds, profile it: container startup? migrations? Respawn? Optimize the biggest cost.

## Self-check

- [ ] My integration tests run against real PostgreSQL and Redis, locally and in CI.
- [ ] Authorization's negative paths are tested.
- [ ] A repeated concurrent test proves seat holds are race-safe.

## Further reading

- Microsoft Learn: *Integration tests in ASP.NET Core*.
- The Testcontainers for .NET docs: PostgreSQL module, best practices.
- The Respawn README (Jimmy Bogard).
