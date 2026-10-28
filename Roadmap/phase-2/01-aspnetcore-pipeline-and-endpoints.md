# 1. Request Pipeline, Routing, Minimal APIs and Controllers

> Phase 2 · Week 1 · [← Phase index](README.md)

**Why this matters:** Every ASP.NET Core bug that sounds like "CORS doesn't work", "auth is ignored", "my exception returns HTML" or "the header is missing" is really a **pipeline order** bug. If you understand the pipeline, you can reason about *any* request.

---

## 1.1 The life of a request

```
Client ──HTTP──► Reverse proxy (optional: ingress / YARP / Nginx / Azure Front Door)
                  │  TLS termination, load balancing
                  ▼
               Kestrel (the built-in, cross-platform web server)
                  │  parses HTTP/1.1, HTTP/2 or HTTP/3 into an HttpContext
                  ▼
        ┌──────── Middleware pipeline (a chain of RequestDelegates) ────────┐
        │ ExceptionHandler → HSTS → HttpsRedirection → Routing → CORS →     │
        │ Authentication → Authorization → RateLimiter → OutputCache → ...  │
        └───────────────────────────────┬────────────────────────────────────┘
                                        ▼
                              Endpoint (your handler / controller action)
                                        │
                     the response flows back through the same middleware in reverse
```

- **`HttpContext`** holds everything for one request: `Request`, `Response`, `User`, `RequestServices` (the DI scope), `RequestAborted` (a cancellation token) and `TraceIdentifier`.
- **One request = one DI scope.** Scoped services, such as `DbContext`, live for exactly one request.
- `HttpContext` **isn't thread-safe**, and it must not be used after the request completes. Never capture it in background work.

---

## 1.2 Kestrel, reverse proxies and forwarded headers

- In containers, .NET 8+ apps listen on port **8080** by default (`ASPNETCORE_HTTP_PORTS`).
- Behind a proxy, the app sees the *proxy's* IP address and `http`. Use **Forwarded Headers** to restore the real client IP and scheme. Otherwise rate limiting, logging and redirects all break.

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownProxies.Clear();          // in production, list your proxy IPs or networks instead of trusting everyone
    o.KnownNetworks.Clear();
});

app.UseForwardedHeaders();           // very first middleware
```

- Kestrel limits protect you: `MaxRequestBodySize` (30 MB by default), header size and request timeouts. Lower them for public APIs.

---

## 1.3 Middleware

Middleware is a function that receives the `HttpContext` and a `next` delegate. It can act **before** and **after** the rest of the pipeline, or **short-circuit** it.

```csharp
// Inline middleware
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    await next(context);                       // call the rest of the pipeline
});

// Class-based middleware (preferred for anything non-trivial)
public sealed class RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)   // inject SCOPED services here as parameters, not in the ctor
    {
        long start = Stopwatch.GetTimestamp();
        try
        {
            await next(context);
        }
        finally
        {
            var elapsed = Stopwatch.GetElapsedTime(start);
            if (elapsed > TimeSpan.FromMilliseconds(500))
                logger.LogWarning("Slow request {Method} {Path} took {ElapsedMs:F0} ms, status {StatusCode}",
                    context.Request.Method, context.Request.Path, elapsed.TotalMilliseconds, context.Response.StatusCode);
        }
    }
}

app.UseMiddleware<RequestTimingMiddleware>();
```

> ⚠️ Convention-based middleware is created **once** for the whole app, so it behaves like a singleton. If you inject a scoped service such as `DbContext` into its *constructor*, you get a captive dependency. Inject scoped services as `InvokeAsync` parameters instead.

### The recommended order (and why)

```csharp
var app = builder.Build();

app.UseForwardedHeaders();          // 1. fix scheme and IP before anything uses them
app.UseExceptionHandler();          // 2. outermost, so it catches exceptions from everything below
if (!app.Environment.IsDevelopment())
    app.UseHsts();                  // 3. tell browsers "HTTPS only"
app.UseHttpsRedirection();          // 4.
app.UseResponseCompression();       // 5. before anything that writes the response body
app.UseRouting();                   // 6. selects the endpoint; later middleware can read its metadata
app.UseCors();                      // 7. after routing (endpoint policies), before auth (preflight has no token)
app.UseAuthentication();            // 8. who are you? sets HttpContext.User
app.UseAuthorization();             // 9. are you allowed? reads the endpoint's [Authorize] metadata
app.UseRateLimiter();               // 10. after auth, so you can partition by user
app.UseOutputCache();               // 11. after CORS and auth
app.UseMiddleware<RequestTimingMiddleware>();

app.MapEventEndpoints();            // endpoints run at the end of the pipeline
app.Run();
```

| Common bug | Cause |
|---|---|
| `[Authorize]` is ignored | `UseAuthorization` isn't called, or it runs before `UseRouting` (no endpoint metadata yet) |
| The CORS preflight fails with 401 | `UseCors` runs after authentication, or the policy isn't applied |
| An exception returns an empty 500 or a developer page in production | `UseExceptionHandler` is missing, or it isn't the first middleware |
| The client IP is always the proxy's | `UseForwardedHeaders` is missing or placed too late |

> `WebApplication` adds `UseRouting` and the endpoint execution automatically if you don't call them. Call `UseRouting` **explicitly** whenever middleware such as CORS, auth or rate limiting must run *between* routing and the endpoint.

---

## 1.4 Routing

```csharp
app.MapGet("/api/events/{id:guid}", ...);           // route constraint: only matches GUIDs
app.MapGet("/api/events/{slug:regex(^[a-z0-9-]+$)}", ...);
app.MapGet("/api/files/{**path}", ...);             // catch-all

var events = app.MapGroup("/api/events")           // route groups share a prefix and metadata
    .WithTags("Events")
    .RequireAuthorization();                        // applies to every endpoint in the group
```

- Constraints are for **disambiguating routes**, not for validation. A non-matching GUID gives a 404, not a 400.
- Use **link generation** instead of hard-coded URLs: `.WithName("GetEvent")` together with `TypedResults.CreatedAtRoute(...)` or `LinkGenerator`.

---

## 1.5 Minimal APIs

### Parameter binding: where values come from

| Source | Example | Attribute |
|---|---|---|
| Route | `/events/{id}` → `Guid id` | inferred, or `[FromRoute]` |
| Query string | `?page=2` → `int page` | inferred for simple types, or `[FromQuery]` |
| Header | `[FromHeader(Name = "Idempotency-Key")] string? key` | required |
| Body (JSON) | a complex type such as `CreateEventRequest request` | inferred, or `[FromBody]` |
| DI services | `TicketHubDbContext db` | inferred, or `[FromServices]` |
| Special types | `HttpContext`, `HttpRequest`, `CancellationToken`, `ClaimsPrincipal` | automatic |
| Group of parameters | `[AsParameters] EventQuery query` (a class or record with its own binding attributes) | `[AsParameters]` |

### Typed results: compile-checked responses that are visible in OpenAPI

```csharp
public static class EventEndpoints
{
    public static IEndpointRouteBuilder MapEventEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/events").WithTags("Events");

        group.MapGet("/{id:guid}", GetEvent).WithName("GetEvent");
        group.MapPost("/", CreateEvent).RequireAuthorization("Organizer");

        return app;
    }

    private static async Task<Results<Ok<EventDto>, NotFound>> GetEvent(
        Guid id, TicketHubDbContext db, CancellationToken ct)
    {
        var dto = await db.Events
            .AsNoTracking()
            .Where(e => e.Id == id)
            .Select(e => new EventDto(e.Id, e.Name, e.StartsAt, e.Status.ToString()))
            .FirstOrDefaultAsync(ct);

        return dto is null ? TypedResults.NotFound() : TypedResults.Ok(dto);
    }

    private static async Task<Created<EventDto>> CreateEvent(
        CreateEventRequest request, TicketHubDbContext db, CancellationToken ct)
    {
        var evt = Event.Create(request.Name, request.VenueName, request.StartsAt, request.Price, request.Currency);
        db.Events.Add(evt);
        await db.SaveChangesAsync(ct);

        var dto = new EventDto(evt.Id, evt.Name, evt.StartsAt, evt.Status.ToString());
        return TypedResults.Created($"/api/events/{evt.Id}", dto);
    }
}
```

- `Results<Ok<EventDto>, NotFound>` is a **union return type**: the compiler checks that you only return those results, and OpenAPI documents both status codes automatically.
- **Organize endpoints by feature** (`EventEndpoints`, `HoldEndpoints`, `OrderEndpoints`), not into one giant `Program.cs`.
- Use **named methods** rather than lambdas for anything longer than a few lines. They're easier to test and read, and XML comments flow into OpenAPI.

### Endpoint filters: the Minimal API version of action filters

```csharp
group.AddEndpointFilter(async (ctx, next) =>
{
    var logger = ctx.HttpContext.RequestServices.GetRequiredService<ILoggerFactory>().CreateLogger("Audit");
    logger.LogInformation("User {User} called {Path}", ctx.HttpContext.User.Identity?.Name, ctx.HttpContext.Request.Path);
    return await next(ctx);
});
```

---

## 1.6 Controllers

```csharp
[ApiController]
[Route("api/orders")]
public sealed class OrdersController(IOrderQueries queries) : ControllerBase
{
    [HttpGet("{id:guid}")]
    [ProducesResponseType<OrderDto>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> Get(Guid id, CancellationToken ct)
    {
        var order = await queries.FindAsync(id, ct);
        return order is null ? NotFound() : order;
    }
}

builder.Services.AddControllers();
app.MapControllers();
```

**What `[ApiController]` gives you:** an automatic **400 with `ValidationProblemDetails`** when model state is invalid, binding-source inference, and ProblemDetails for error status codes.

**Filters** (authorization → resource → action → exception → result) are powerful, but they're MVC-only.

| Choose Minimal APIs when… | Choose controllers when… |
|---|---|
| You're starting a new service or prefer small feature-focused files | The existing codebase uses controllers (very common in enterprise) |
| Performance or AOT matters (Minimal APIs are lighter and AOT-friendly) | You rely on MVC filters, conventions or model-binding extensibility |
| You want a vertical-slice organization | The team is more productive with them |

> Know **both**. Interviews and real jobs mix them. TicketHub uses Minimal APIs.

---

## 1.7 JSON configuration

```csharp
builder.Services.ConfigureHttpJsonOptions(o =>
{
    o.SerializerOptions.Converters.Add(new JsonStringEnumConverter());     // enums as "Published", not 1
    o.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
});
```

The defaults are camelCase property names and case-insensitive reading. Keep the defaults unless you have a contract reason to change them, because every client depends on them.

---

## Interview questions

<details><summary><b>1. Describe the order of the main middleware and why it matters.</b></summary>

ForwardedHeaders → ExceptionHandler (outermost, so it catches everything) → HSTS and HttpsRedirection → Routing (selects the endpoint so later middleware can read its metadata) → CORS (before auth, because preflight requests carry no token) → Authentication (sets `User`) → Authorization (checks the endpoint's policies) → rate limiting and output caching → endpoints. Each middleware only sees what earlier middleware prepared.
</details>

<details><summary><b>2. How do you use a scoped service inside custom middleware?</b></summary>

Middleware is constructed once, so injecting a scoped service in its constructor makes it captive. Inject scoped services as parameters of `InvokeAsync`, which are resolved per request from `HttpContext.RequestServices`. Alternatively, implement `IMiddleware`, which is resolved from DI per request.
</details>

<details><summary><b>3. Minimal APIs vs controllers?</b></summary>

Both run on the same routing and hosting. Minimal APIs are lighter, AOT-friendly and feature-oriented, and they use endpoint filters and `TypedResults`. Controllers offer the MVC filter pipeline, conventions and `[ApiController]` behaviors. Choose based on the codebase and the team, and know both.
</details>

<details><summary><b>4. What does <code>[ApiController]</code> do?</b></summary>

It returns an automatic 400 `ValidationProblemDetails` for invalid model state, infers binding sources, requires attribute routing, and produces ProblemDetails for error responses.
</details>

<details><summary><b>5. Why can't you use <code>HttpContext</code> in a background task started from a request?</b></summary>

`HttpContext` and its scoped services are recycled or disposed when the request ends, and `HttpContext` isn't thread-safe. Copy the data you need (IDs, user ID) and queue the work instead.
</details>

---

## Exercises

1. **Order experiments:** move `UseCors` after `UseAuthentication`, send a preflight `OPTIONS` request from a `.http` file, and observe the effect. Do the same with `UseExceptionHandler` placed last. Write down each failure mode.
2. **Middleware:** write a `CorrelationIdMiddleware` that reads `X-Correlation-Id` (or creates one), adds it to the response and to a logging scope, and test it.
3. **Both styles:** implement `GET /api/events/{id}` once with Minimal APIs and once with a controller. Compare the generated OpenAPI documents.
4. **Read the source:** open `src/Http/Routing/src/EndpointMiddleware.cs` in the dotnet/aspnetcore repository and explain in 5 sentences what it does.

## Self-check

- [ ] I can draw the pipeline and explain every middleware's position.
- [ ] I know the binding sources and use `TypedResults` with union return types.
- [ ] I organize endpoints by feature with route groups.
- [ ] I can explain the trade-offs between Minimal APIs and controllers.

## Further reading

- Microsoft Learn: *ASP.NET Core middleware* (and its middleware order diagram), *Routing*, *Minimal APIs overview*, *Parameter binding*, *Create responses in Minimal API apps*.
- *ASP.NET Core in Action* (Andrew Lock), Part 1.
