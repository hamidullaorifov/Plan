# 2. Validation, Errors (ProblemDetails) and OpenAPI

> Phase 2 · Week 2 · [← Phase index](README.md)

**Why this matters:** Clients integrate against your *contract*: the shape of requests, responses and errors. Inconsistent error formats, missing validation and undocumented endpoints generate support tickets and integration bugs. A Middle developer ships APIs whose failures are as well-designed as their successes.

---

## 2.1 Two kinds of validation

| | **Input validation** | **Business rules** |
|---|---|---|
| Question | "Is this request well-formed?" | "Is this allowed right now?" |
| Examples | a required name, max length, a valid email, between 1 and 10 seat codes | the seat is already held, the event isn't on sale, the hold has expired |
| Where | the API boundary (attributes, validators) | the domain (`Result` / errors from [Phase 1, guide 7](../phase-1/07-error-handling.md)) |
| Response | **400** with field errors | **409 / 422 / 404**, depending on the rule |

Don't put business rules in attributes, and don't put shape validation deep in the domain.

---

## 2.2 Built-in validation for Minimal APIs (.NET 10)

```csharp
builder.Services.AddValidation();   // a source-generated validator; validates query, header and body parameters
```

```csharp
using System.ComponentModel.DataAnnotations;

public sealed record CreateEventRequest(
    [Required, StringLength(200, MinimumLength = 3)] string Name,
    [Required, StringLength(200)] string VenueName,
    DateTimeOffset StartsAt,
    [Range(1, 100_000)] int Capacity);

public sealed record HoldSeatsRequest(
    [Required, MinLength(1), MaxLength(10)] string[] SeatCodes);
```

When validation fails, the endpoint returns **400** with a `ValidationProblemDetails` body:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": { "Name": ["The field Name must be a string with a minimum length of 3 and a maximum length of 200."] }
}
```

- Nested objects, collections and `IValidatableObject` are supported.
- Opt out per endpoint with `.DisableValidation()`.
- The source generator only discovers types in the assembly where `AddValidation()` is called.
- For cross-field rules ("end date after start date"), implement `IValidatableObject` or use **FluentValidation**, which is popular in enterprise code and Apache-2.0 licensed. FluentValidation's automatic ASP.NET integration is deprecated, so call validators explicitly or from an endpoint filter.

---

## 2.3 ProblemDetails: one error format for everything

[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) defines a standard JSON error body (`type`, `title`, `status`, `detail`, `instance` plus extensions). Use it for **every** error: validation, domain, authorization and unexpected failures.

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Instance = $"{ctx.HttpContext.Request.Method} {ctx.HttpContext.Request.Path}";
        ctx.ProblemDetails.Extensions["traceId"] = Activity.Current?.Id ?? ctx.HttpContext.TraceIdentifier;
    };
});

app.UseExceptionHandler();   // uses IProblemDetailsService and the registered IExceptionHandlers
app.UseStatusCodePages();    // turns empty 404/405 responses into ProblemDetails
```

### A global exception handler for unexpected failures

```csharp
public sealed class GlobalExceptionHandler(IProblemDetailsService problemDetails, ILogger<GlobalExceptionHandler> logger)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken ct)
    {
        if (exception is OperationCanceledException && httpContext.RequestAborted.IsCancellationRequested)
        {
            httpContext.Response.StatusCode = 499;            // the client went away; not our error
            return true;
        }

        logger.LogError(exception, "Unhandled exception for {Method} {Path}", httpContext.Request.Method, httpContext.Request.Path);

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails =
            {
                Title = "An unexpected error occurred.",
                Detail = "Use the traceId when contacting support.",   // never the exception message or stack trace
                Status = StatusCodes.Status500InternalServerError,
            },
        });
    }
}

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

> Since .NET 10, the exception-handler middleware **doesn't log** exceptions that an `IExceptionHandler` handled. Log in your handler, as shown above.

### Mapping domain errors to HTTP

Use the `Error` type from Phase 1 with a single mapping point, so every endpoint maps errors consistently:

```csharp
public static class ErrorHttpMapping
{
    public static ProblemHttpResult ToProblem(this Error error) => TypedResults.Problem(
        title: error.Code,
        detail: error.Message,
        statusCode: error.Code switch
        {
            _ when error.Code.EndsWith(".NotFound", StringComparison.Ordinal) => StatusCodes.Status404NotFound,
            "Event.SeatsUnavailable" or "Hold.Expired"                         => StatusCodes.Status409Conflict,
            "Event.NotOnSale"                                                  => StatusCodes.Status409Conflict,
            _                                                                  => StatusCodes.Status400BadRequest,
        },
        extensions: new Dictionary<string, object?> { ["code"] = error.Code });
}

// Endpoint
var result = await holds.HoldAsync(eventId, request.SeatCodes, userId, ct);
return result.IsSuccess
    ? TypedResults.Created($"/api/holds/{result.Value.Id}", result.Value)
    : result.Error!.ToProblem();
```

| Status | Use for |
|---|---|
| 400 | malformed input, validation errors |
| 401 | not authenticated (missing or invalid token) |
| 403 | authenticated but not allowed |
| 404 | the resource doesn't exist, or **the caller isn't allowed to know it exists** |
| 409 | conflicts with the current state (seat taken, concurrency conflict) |
| 412 | `If-Match` precondition failed (ETag mismatch) |
| 422 | well-formed but semantically invalid (some teams use 400 for this instead; pick one convention) |
| 429 | rate limited |
| 500 | unexpected failure (a bug) |
| 503 | a dependency is down or the service is overloaded (temporary) |

---

## 2.4 OpenAPI (built in since .NET 9; OpenAPI 3.1 by default in .NET 10)

```csharp
builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((document, context, ct) =>
    {
        document.Info.Title = "TicketHub API";
        document.Info.Version = "v1";
        return Task.CompletedTask;
    });
});

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                     // /openapi/v1.json (or .yaml)
    app.MapScalarApiReference();          // package Scalar.AspNetCore: interactive UI at /scalar
}
```

- Swashbuckle is **no longer in the templates**. Its UI package still works (`app.UseSwaggerUI(o => o.SwaggerEndpoint("/openapi/v1.json", "v1"))`).
- Enrich operations with `.WithSummary("...")`, `.WithDescription("...")`, `.Produces<T>(201)` and `.ProducesProblem(409)`, or with XML doc comments on named handler methods (set `<GenerateDocumentationFile>true</GenerateDocumentationFile>`).
- **Contract checks in CI:** generate the document at build time (`<OpenApiGenerateDocumentsOnBuild>true</OpenApiGenerateDocumentsOnBuild>`), commit it, and fail the PR on breaking diffs, for example with `oasdiff`.
- **Generating clients:** Kiota or NSwag generate typed C# or TypeScript clients from the document. That's how front-end teams consume your API safely.

> OpenAPI 3.1 changed the `Microsoft.OpenApi` library to v2. Custom transformers written for .NET 9 need small updates. For example, `OpenApiAny` is replaced by `JsonNode`.

---

## Interview questions

<details><summary><b>1. Input validation vs business rules: where does each belong?</b></summary>

Shape and format validation happens at the API boundary (attributes or validators) and returns 400 with field errors. Business rules live in the domain and return domain errors, which are mapped to 409/422/404. Mixing them leads to duplicated or missing rules.
</details>

<details><summary><b>2. What is ProblemDetails and why use it?</b></summary>

The RFC 9457 standard JSON error format. One consistent shape (`type`, `title`, `status`, `detail` plus extensions such as `traceId` and `code`) lets clients and tools handle errors generically. ASP.NET Core supports it through `AddProblemDetails`, `IProblemDetailsService` and `IExceptionHandler`.
</details>

<details><summary><b>3. What should a 500 response contain?</b></summary>

A generic message and a correlation/trace ID, and nothing else. Exposing exception messages or stack traces leaks implementation details, which is an OWASP finding. Log the full exception server-side.
</details>

<details><summary><b>4. 401 vs 403 vs 404 for a resource the user can't access?</b></summary>

401 means they aren't authenticated. 403 means they're authenticated but forbidden. 404 is preferable when even revealing that the resource exists is a leak, for example another customer's order ID.
</details>

---

## Exercises

1. Add validation to every TicketHub request DTO, and write `.http` requests that trigger each rule.
2. Implement `GlobalExceptionHandler` and an `ErrorHttpMapping`. Throw an exception from a test endpoint and verify that the response contains a `traceId` and no stack trace.
3. Generate a TypeScript client with Kiota from your OpenAPI document. Is the contract clear enough for a front-end developer who has never talked to you?
4. Add build-time OpenAPI generation and compare two versions of the document after making a breaking change, such as renaming a property.

## Self-check

- [ ] Every error from my API is ProblemDetails, including validation errors and 500s.
- [ ] I separate input validation from business rules.
- [ ] My OpenAPI document is accurate (status codes, schemas) and explorable through Scalar.

## Further reading

- Microsoft Learn: *Handle errors in ASP.NET Core APIs*, *Validation in Minimal APIs*, *Generate OpenAPI documents*, *Customize OpenAPI documents*.
- RFC 9457, *Problem Details for HTTP APIs*.
