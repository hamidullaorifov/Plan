# 8. Application Security: OWASP, Secrets and the Supply Chain

> Phase 4 · Weeks 10–11 · [← Phase index](README.md)
> Week 10 covers sections 8.1–8.5; week 11 covers 8.6–8.10.

**Why this matters:** Security bugs are the most expensive bugs. AI-generated code adds new risks: in the 2026 Stack Overflow survey, 24% of developers lost time to AI-caused security or privacy issues and 45% to hallucinated packages or configuration, while only 23% used security tooling at all. A Middle developer must prevent the common vulnerabilities *by default* and recognize them in code review.

---

## 8.1 The OWASP Top 10, mapped to ASP.NET Core

The OWASP Top 10 is the industry's list of the most critical web application risks. The current edition (2025) is summarized below; check owasp.org for the official wording and order.

| Risk | What it looks like in .NET | Defense |
|---|---|---|
| **Broken access control** (#1 for years; includes SSRF in the 2025 edition) | `GET /api/orders/{id}` returns *any* user's order (IDOR); admin endpoints without policies | authorization policies on every endpoint; **resource-based checks** (owner == current user); deny by default |
| **Security misconfiguration** | detailed errors in production, CORS `AllowAnyOrigin` with credentials, Swagger exposed publicly, default credentials | environment-specific configuration, ProblemDetails without stack traces, explicit CORS origins, hardened headers |
| **Software supply chain failures** | vulnerable or malicious NuGet packages, compromised CI actions | NuGet audit, lock files, Dependabot/Renovate, pinned actions, SBOMs (8.7) |
| **Cryptographic failures** | homemade password hashing, secrets in code, HTTP instead of HTTPS | ASP.NET Core Identity / `PasswordHasher`, the Data Protection API, TLS everywhere, HSTS |
| **Injection** | string-concatenated SQL, `FromSqlRaw($"... {input}")`, command injection, log injection | parameterized queries (EF Core LINQ, `FromSql` with interpolation, Dapper parameters), no shelling out with user input |
| **Insecure design** | no rate limit on login or seat holds, enumerable IDs, business-logic abuse | threat modeling (8.8), rate limiting, abuse cases in design docs |
| **Authentication failures** | long-lived tokens, missing audience/issuer validation, weak password policies | OIDC providers, short-lived access tokens, proper JWT validation (Phase 2), MFA |
| **Software and data integrity failures** | insecure deserialization (`BinaryFormatter`, `TypeNameHandling.All`), unsigned updates | System.Text.Json without polymorphic type names from input; signed artifacts |
| **Logging and alerting failures** | no audit trail for logins or permission changes; no alerts | security event logging (without secrets), alerts on anomalies |
| **Mishandling of exceptional conditions** | fail-open authorization on errors; leaking internals in exceptions | fail closed; a global exception handler with generic messages |

---

## 8.2 Broken access control: the IDOR bug

```csharp
// ❌ IDOR: any authenticated user can read any order by guessing or iterating IDs
app.MapGet("/api/orders/{id:guid}", async (Guid id, ShopDbContext db, CancellationToken ct) =>
    await db.Orders.FindAsync([id], ct) is { } order ? Results.Ok(order.ToDto()) : Results.NotFound())
   .RequireAuthorization();

// ✅ Scope the query to the current user (and return 404, so you don't reveal that the order exists)
app.MapGet("/api/orders/{id:guid}", async (Guid id, ClaimsPrincipal user, ShopDbContext db, CancellationToken ct) =>
{
    var customerId = user.GetCustomerId();
    var order = await db.Orders.AsNoTracking()
        .Where(o => o.Id == id && o.CustomerId == customerId)
        .Select(o => o.ToDto())
        .SingleOrDefaultAsync(ct);
    return order is null ? Results.NotFound() : Results.Ok(order);
}).RequireAuthorization();
```

**Review rule:** for every endpoint that takes an ID, ask *"what stops user A from using user B's ID?"*. GUIDs make guessing harder but are **not** authorization.

**Mass assignment** is the sibling bug: binding a request directly to an entity lets a client set `IsAdmin` or `Price`. Always bind to **request DTOs** that contain only the allowed fields.

---

## 8.3 Injection

```csharp
// ❌ SQL injection: the input becomes part of the SQL text
var orders = db.Orders.FromSqlRaw($"SELECT * FROM orders WHERE email = '{email}'");

// ✅ FromSql with an interpolated FormattableString → parameters, not concatenation
var safe = db.Orders.FromSql($"SELECT * FROM orders WHERE email = {email}");

// ✅ Dapper: always pass parameters
var rows = await connection.QueryAsync<OrderDto>("SELECT … WHERE email = @email", new { email });
```

- **Dynamic sort and filter columns** can't be parameters. Map user input to an **allow-list** of known column names.
- **XSS:** Razor encodes output by default. Danger points are `Html.Raw`, `MarkupString` (Blazor), building HTML strings, and SPA frameworks' "dangerously set HTML" APIs. Add a **Content Security Policy**.
- **Log injection / log forging:** structured logging keeps values as fields; never log raw user input as the message template.
- **Path traversal:** never build file paths from user input without normalizing them and checking they stay inside an allowed directory.

---

## 8.4 SSRF (server-side request forgery)

A feature like "import an image from a URL" lets attackers make *your server* call internal addresses, such as the cloud metadata endpoint (`169.254.169.254`) or internal admin services.

**Defenses:**

- an **allow-list** of hosts where possible;
- resolve DNS and **block private, loopback, link-local and metadata IP ranges** (re-check after redirects, or disable redirects);
- network egress restrictions;
- no raw response bodies or error details echoed back to the user.

---

## 8.5 Hardening ASP.NET Core

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler();   // ProblemDetails, no stack traces
    app.UseHsts();               // Strict-Transport-Security
}
app.UseHttpsRedirection();

app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;
    headers["X-Content-Type-Options"] = "nosniff";
    headers["Referrer-Policy"] = "no-referrer";
    headers["X-Frame-Options"] = "DENY";
    headers["Content-Security-Policy"] = "default-src 'self'; frame-ancestors 'none'";   // tune per app
    await next();
});

builder.Services.AddCors(o => o.AddPolicy("web", p => p
    .WithOrigins("https://app.shop.example")   // explicit origins, never AllowAnyOrigin together with credentials
    .AllowAnyHeader()
    .WithMethods("GET", "POST", "PUT", "DELETE")));
```

| Item | Practice |
|---|---|
| OpenAPI/Scalar UI | Development only, or behind authentication |
| Cookies | `Secure`, `HttpOnly`, `SameSite=Lax/Strict`; antiforgery for cookie-authenticated form posts |
| **Data Protection keys** | in multi-instance deployments, **persist and protect them** (Blob Storage + Key Vault, a database, Redis). Otherwise cookies and antiforgery tokens fail across instances and restarts |
| Request limits | body size limits, rate limiting (Phase 2), timeouts |
| Error responses | generic message plus `traceId`; details only in logs |

```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("shop")
    .PersistKeysToAzureBlobStorage(new Uri(builder.Configuration["DataProtection:BlobUri"]!), credential)
    .ProtectKeysWithAzureKeyVault(new Uri(builder.Configuration["DataProtection:KeyUri"]!), credential);
```

---

## 8.6 Secrets management

| ❌ Never | ✅ Instead |
|---|---|
| secrets in `appsettings*.json`, code, Dockerfiles, Git history | user secrets (local), Key Vault / Secrets Manager (deployed) |
| long-lived cloud keys in CI | OIDC federation (GitHub → Azure/AWS) |
| shared passwords between environments | per-environment secrets and identities |
| connection strings with passwords where avoidable | managed identity / IAM database authentication |

**If a secret leaks** (committed, pasted into a chat or into an AI prompt, printed in logs):

1. **Rotate it immediately.** Removing it from Git isn't enough, because history, forks and caches keep it.
2. Check the access logs for misuse.
3. Then purge it from history if your policy requires it, and add **secret scanning** (GitHub secret scanning with push protection, gitleaks).

---

## 8.7 The software supply chain

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
  <NuGetAudit>true</NuGetAudit>                     <!-- on by default in modern SDKs -->
  <NuGetAuditMode>all</NuGetAuditMode>              <!-- include transitive packages -->
  <NuGetAuditLevel>moderate</NuGetAuditLevel>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

```powershell
dotnet list package --vulnerable --include-transitive   # also: dotnet package list --vulnerable (newer SDK syntax)
dotnet list package --outdated
```

| Practice | Why |
|---|---|
| **NuGet audit + `TreatWarningsAsErrors`** | builds fail on known-vulnerable packages (warnings NU1901–NU1904) |
| **Central package management** (`Directory.Packages.props`) | one version per package across the solution |
| **Lock files** + `dotnet restore --locked-mode` in CI | reproducible restores; no surprise upgrades |
| **Dependabot / Renovate** | automated update PRs |
| **Package source mapping** (`nuget.config`) | prevents dependency-confusion attacks between public and private feeds |
| `--min-publish-age-hours` on package sources (.NET 11 SDK) | avoid freshly published, possibly malicious versions |
| **Verify every AI-suggested package** | check it exists, is maintained, has real downloads and the expected owner. Typosquats and hallucinated names are an active attack vector |
| **SBOM** (CycloneDX, Microsoft SBOM tool) | know what you ship; respond quickly to the next Log4Shell-style event |
| Pin GitHub Actions to commit SHAs | prevent a compromised tag from running in your pipeline |
| Scan images (Trivy) and keep base images patched | OS-level CVEs |

---

## 8.8 Threat modeling in 30 minutes (STRIDE)

For a new feature, draw the data flow (client → API → database → third party), then ask, for each element:

| Threat | Question | Example mitigation |
|---|---|---|
| **S**poofing | can someone pretend to be another user or service? | OIDC, validated webhook signatures |
| **T**ampering | can data be modified in transit or at rest? | TLS, signatures, concurrency tokens |
| **R**epudiation | can someone deny an action? | audit logs |
| **I**nformation disclosure | can data leak? | authorization checks, no PII in logs, encryption |
| **D**enial of service | can someone exhaust resources? | rate limits, timeouts, request size limits |
| **E**levation of privilege | can a user gain more rights? | least privilege, no mass assignment |

Write the findings into the design doc's security section (Phase 3, guide 9).

---

## 8.9 Security in code review: a checklist

- [ ] Every endpoint has an authorization requirement, and resource ownership is checked.
- [ ] Inputs are bound to DTOs, validated, and size-limited.
- [ ] There's no SQL or command built by string concatenation with input.
- [ ] No secrets, tokens or PII appear in code, logs, exceptions or test data.
- [ ] Outgoing requests to user-supplied URLs are restricted (SSRF).
- [ ] New packages are justified, maintained and license-checked; AI-suggested packages are verified.
- [ ] Error responses don't leak internals.
- [ ] Webhooks are signature-validated and replay-protected; uploaded files are type- and size-checked and stored outside the web root.

---

## 8.10 AI-specific security notes (preview of Phase 5)

- Never paste secrets, customer data or proprietary code into AI tools your company hasn't approved.
- AI-generated code needs the **same review** as human code, plus extra scrutiny of authorization, input validation, crypto and dependencies.
- Apps that *call* LLMs face prompt injection and data exfiltration through tools. That's covered in Phase 5, guide 8.

---

## Interview questions

<details><summary><b>1. What is IDOR, and how do you prevent it?</b></summary>

Insecure direct object reference: an endpoint returns or modifies a resource by ID without checking that the caller may access it. Prevent it with resource-based authorization, by scoping queries to the current user or tenant, and by testing with two users. Unguessable IDs help but aren't authorization.
</details>

<details><summary><b>2. Is `FromSqlRaw` safe? What about `FromSql`?</b></summary>

`FromSqlRaw` with concatenated or interpolated input is injectable. `FromSql` (or `FromSqlInterpolated`) converts interpolated values into parameters. Identifiers such as column names can't be parameters, so use an allow-list.
</details>

<details><summary><b>3. A secret was committed to Git. What do you do?</b></summary>

Rotate or revoke it immediately, then check for misuse. Removing it from the repository afterward is cleanup: history, clones and caches still have it. Add secret scanning with push protection to prevent a recurrence.
</details>

<details><summary><b>4. How do you protect against supply-chain attacks in .NET?</b></summary>

NuGet audit failing the build, lock files and locked-mode restores, central package management, package source mapping, automated update PRs, verifying new packages (especially AI-suggested ones), pinned CI actions, SBOMs and image scanning.
</details>

<details><summary><b>5. Why must Data Protection keys be persisted in multi-instance deployments?</b></summary>

They encrypt cookies, antiforgery tokens and other protected payloads. If each instance generates its own keys, or loses them on restart, payloads created by one instance can't be read by another. Users get logged out and forms fail. Persist the keys centrally and protect them with a key vault.
</details>

---

## Exercises

1. **Break and fix:** add an IDOR, an SQL injection (`FromSqlRaw`), an SSRF-prone URL import and a verbose error page to a lab API. Exploit each with Bruno or curl, then fix them ([Lab 10](10-labs.md#lab-10-break-and-fix-owasp)).
2. **Supply chain:** enable NuGet audit in transitive mode, lock files, central package management and package source mapping in a lab solution ([Lab 11](10-labs.md#lab-11-supply-chain-hardening)).
3. Run a STRIDE session for one feature at work and add the findings to its design doc.
4. Turn on secret scanning with push protection in your GitHub repositories.

## Self-check

- [ ] I can map the OWASP Top 10 risks to concrete ASP.NET Core defenses.
- [ ] I prevent IDOR, mass assignment, injection and SSRF by default.
- [ ] I harden headers, errors, CORS, cookies and Data Protection.
- [ ] I keep secrets out of code and know the leak response.
- [ ] I secure the dependency supply chain and verify AI-suggested packages.

## Further reading

- OWASP: *Top 10* (owasp.org), *ASVS*, *Cheat Sheet Series* (especially the .NET security cheat sheet).
- Microsoft Learn: *ASP.NET Core security overview*, *Data Protection*, *Safe storage of app secrets*, *NuGet Audit*.
- Adam Shostack, *Threat Modeling: Designing for Security*.
