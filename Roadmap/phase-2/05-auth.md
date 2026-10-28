# 5. Authentication and Authorization: JWT, OAuth2/OIDC and Policies

> Phase 2 · Weeks 7–8 · [← Phase index](README.md)

**Why this matters:** Broken access control is the **#1 web security risk** (OWASP). Auth is also the topic where juniors most often copy-paste code they don't understand. A Middle developer can explain a token flow end to end and design authorization that holds up when an attacker changes an ID in the URL.

---

## 5.1 Vocabulary

| Term | Meaning |
|---|---|
| **Authentication (AuthN)** | *Who are you?* It produces a `ClaimsPrincipal` (`HttpContext.User`) |
| **Authorization (AuthZ)** | *What may you do?* It evaluates policies, roles and ownership |
| **Claim** | a statement about the user: `sub`, `email`, `role`, `tenant_id` |
| **Identity Provider (IdP)** | the system that authenticates users and issues tokens: Keycloak, Entra ID, Auth0, Okta |
| **OAuth 2.0** | a framework for **delegated authorization**: getting an *access token* for an API |
| **OpenID Connect (OIDC)** | an **identity layer** on top of OAuth 2.0 that adds the *ID token* and the user-info endpoint |
| **Access token** | sent to APIs (`Authorization: Bearer ...`); short-lived (5–60 min) |
| **ID token** | for the *client app*, to learn who logged in. **Never** send it to APIs |
| **Refresh token** | long-lived; exchanged for new access tokens; must be stored securely |

---

## 5.2 Cookies vs bearer tokens

| | Cookie authentication | JWT bearer |
|---|---|---|
| Typical client | server-rendered web apps, BFF (backend-for-frontend) | APIs called by SPAs (through a BFF), mobile apps and other services |
| Sent | automatically by the browser | explicitly in the `Authorization` header |
| Revocation | easy (server-side session) | hard: the token stays valid until it expires, which is why lifetimes are short |
| Main risk | CSRF (mitigated by `SameSite` cookies and anti-forgery tokens) | token theft through XSS if tokens are stored in `localStorage` |

> **Modern SPA guidance:** the **BFF pattern**. The browser holds only an `HttpOnly`, `Secure`, `SameSite` cookie; the BFF server holds the tokens and calls the APIs. Avoid storing access tokens in `localStorage`.

---

## 5.3 Anatomy of a JWT

```
eyJhbGciOiJSUzI1NiIs...   .   eyJzdWIiOiI0MiIsInJvbGVzIjpbImN1c3RvbWVyIl0s...   .   SflKxwRJSMeKKF2QT4fwpM...
       header                                   payload (claims)                                  signature
```

```json
{ "iss": "http://localhost:8080/realms/tickethub", "sub": "f3a1…", "aud": "tickethub-api",
  "exp": 1767225600, "iat": 1767225300, "roles": ["customer"], "email": "ann@example.com" }
```

**What the API validates:**

1. The **signature**, using the IdP's public key from its JWKS endpoint.
2. **`iss`** (the issuer).
3. **`aud`** (the audience: this token is *for this API*).
4. **`exp` / `nbf`** (the lifetime).

A JWT is **signed, not encrypted**, so anyone can read its payload. Never put secrets in it.

---

## 5.4 OAuth 2.0 flows you must know

### Authorization code + PKCE (users in browsers and mobile apps)

```
User        Client app (SPA/BFF/mobile)           Identity Provider (Keycloak)              API
 │  click "Login"  │                                       │                                  │
 │───────────────► │ 1. create code_verifier (random)      │                                  │
 │                 │    code_challenge = SHA256(verifier)  │                                  │
 │                 │ 2. redirect ─────────────────────────►│ /authorize?response_type=code    │
 │                 │                                       │   &client_id&redirect_uri        │
 │                 │                                       │   &scope=openid profile api      │
 │                 │                                       │   &code_challenge&state          │
 │ 3. login + consent ◄───────────────────────────────────►│                                  │
 │                 │ 4. redirect back with ?code=…&state=… │                                  │
 │                 │ 5. POST /token (code + code_verifier) ►│ checks SHA256(verifier)==challenge│
 │                 │ 6. ◄──── access_token, id_token, refresh_token                           │
 │                 │ 7. GET /api/orders  Authorization: Bearer <access_token> ───────────────►│ validates the JWT
```

- **PKCE** stops an attacker who intercepts the authorization code from exchanging it, because they don't have the `code_verifier`.
- **`state`** protects against CSRF on the redirect, and **`nonce`** protects the ID token against replay.
- The **implicit flow** (tokens in the URL fragment) and the **password grant** are **deprecated**. Don't use them.

### Client credentials (service to service, no user)

```
Worker service ── POST /token (client_id + client_secret or a certificate, grant_type=client_credentials) ──► IdP
               ◄── access_token (sub = the service itself) ──
               ── call the API with the Bearer token ──► API
```

### Refresh tokens

The client exchanges a refresh token for a new access token without sending the user to log in again. Use **refresh token rotation**: each use returns a new refresh token and invalidates the old one, which limits the damage if one is stolen.

---

## 5.5 JWT bearer in ASP.NET Core with Keycloak

```yaml
# docker-compose.yml (excerpt): Keycloak for local development
keycloak:
  image: quay.io/keycloak/keycloak:26.0
  command: start-dev --import-realm
  environment:
    KC_BOOTSTRAP_ADMIN_USERNAME: admin
    KC_BOOTSTRAP_ADMIN_PASSWORD: admin          # local development only
  ports: ["8080:8080"]
  volumes: ["./keycloak:/opt/keycloak/data/import"]   # a realm export with clients, roles and test users
```

**In Keycloak** (admin console → realm `tickethub`):

1. Create a client `tickethub-api` (bearer-only, for audience purposes) and a public client `tickethub-web` with PKCE.
2. Create the realm roles `customer`, `organizer` and `admin`, plus test users.
3. Add an **audience mapper** so access tokens contain `aud: tickethub-api`.
4. Add a **"realm roles" mapper** with the token claim name `roles` (multivalued). Keycloak puts roles in `realm_access.roles` by default, which ASP.NET doesn't read.
5. Export the realm to JSON and commit it, so `docker compose up` gives everyone the same setup.

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];   // http://localhost:8080/realms/tickethub
        options.Audience = builder.Configuration["Auth:Audience"];     // tickethub-api
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
        options.MapInboundClaims = false;                              // keep "sub" and "roles" instead of long URI claim types
        options.TokenValidationParameters.NameClaimType = "preferred_username";
        options.TokenValidationParameters.RoleClaimType = "roles";
    });

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("Customer", p => p.RequireRole("customer"))
    .AddPolicy("Organizer", p => p.RequireRole("organizer", "admin"))
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());  // secure by default

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/api/events", ListEvents).AllowAnonymous();             // explicitly public
app.MapPost("/api/events", CreateEvent).RequireAuthorization("Organizer");
```

> **A fallback policy that requires authentication** means a newly added endpoint is **protected by default**. Forgetting `[Authorize]` is one of the most common real-world security bugs.

Read the user's ID from the `sub` claim:

```csharp
public static class ClaimsPrincipalExtensions
{
    public static Guid GetUserId(this ClaimsPrincipal user) =>
        Guid.TryParse(user.FindFirstValue("sub"), out var id)
            ? id
            : throw new InvalidOperationException("Authenticated user has no valid 'sub' claim.");
}
```

---

## 5.6 Authorization models

| Model | Example | ASP.NET Core |
|---|---|---|
| **Role-based** | organizers can create events | `RequireRole("organizer")` |
| **Claims and policy-based** | only users with `email_verified = true` can check out | `p.RequireClaim("email_verified", "true")` or a custom requirement |
| **Resource-based (ownership)** | only the order's owner can view it | `IAuthorizationService` with an `AuthorizationHandler<TRequirement, TResource>` |
| **Scopes (OAuth)** | a third-party app may only read events | `p.RequireClaim("scope", "events.read")` |

### Resource-based authorization: stopping IDOR

**IDOR (Insecure Direct Object Reference):** `GET /api/orders/123` returns order 123 to *any* authenticated user. It's the most common access-control bug.

```csharp
public sealed class SameOwnerRequirement : IAuthorizationRequirement;

public sealed class OrderOwnerHandler : AuthorizationHandler<SameOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, SameOwnerRequirement requirement, Order order)
    {
        if (context.User.IsInRole("admin") ||
            context.User.FindFirstValue("sub") == order.CustomerId.ToString())
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

builder.Services.AddSingleton<IAuthorizationHandler, OrderOwnerHandler>();

// Endpoint
private static async Task<Results<Ok<OrderDto>, NotFound>> GetOrder(
    Guid id, ClaimsPrincipal user, IAuthorizationService authz, TicketHubDbContext db, CancellationToken ct)
{
    var order = await db.Orders.AsNoTracking().FirstOrDefaultAsync(o => o.Id == id, ct);
    if (order is null) return TypedResults.NotFound();

    var result = await authz.AuthorizeAsync(user, order, new SameOwnerRequirement());
    return result.Succeeded ? TypedResults.Ok(OrderDto.From(order)) : TypedResults.NotFound();   // 404: don't reveal existence
}
```

> **The simplest effective defense:** scope the *query* itself to the user, as in `db.Orders.Where(o => o.Id == id && o.CustomerId == userId)`. Then another user's order simply doesn't exist for them.

---

## 5.7 Service-to-service authentication

- Use **client credentials** with short-lived tokens and narrowly scoped audiences.
- In the cloud, prefer **managed identities / workload identity**, which mean no secrets at all (Phase 4).
- Typed HTTP clients can attach tokens through a `DelegatingHandler` that caches the token until shortly before it expires.

---

## 5.8 ASP.NET Core Identity (when *you* are the IdP)

ASP.NET Core Identity manages users, password hashing, lockout, 2FA and **passkeys** (WebAuthn, new in .NET 10) for a single application. Use it for simple apps. For multiple apps, SSO or enterprise needs, use a dedicated IdP (Keycloak, Entra ID, Auth0) and let your APIs **only validate tokens**.

> **Never implement your own password hashing, token signing or crypto.**

---

## 5.9 Testing auth

- **Unit test** authorization handlers with a constructed `ClaimsPrincipal`.
- **Integration tests** swap in a test authentication scheme instead of calling Keycloak ([guide 9](09-integration-testing.md)).
- **Always test the negative cases:** no token → 401, wrong role → 403, another user's resource → 404.

---

## 5.10 Auth security checklist

- [ ] A fallback policy requires authentication; public endpoints are explicitly `AllowAnonymous`.
- [ ] Ownership checks on every resource accessed by ID (IDOR tests exist).
- [ ] Tokens are validated for issuer, audience, lifetime and signature. Never disable validation "to make it work".
- [ ] Access tokens are short-lived; refresh tokens rotate; there are no tokens in `localStorage` for SPAs (use a BFF).
- [ ] HTTPS everywhere; `RequireHttpsMetadata` is true outside development.
- [ ] No secrets in the code or in Git. Client secrets live in user secrets locally and in Key Vault in the cloud.
- [ ] Tokens and PII aren't logged.

---

## Interview questions

<details><summary><b>1. Walk through the authorization code + PKCE flow.</b></summary>

The client generates a `code_verifier` and its SHA-256 `code_challenge`, then redirects to `/authorize` with the challenge and `state`. The user logs in, and the IdP redirects back with a one-time code. The client posts the code plus the `code_verifier` to `/token`. The IdP checks the hash and returns access, ID and refresh tokens, and the client calls APIs with the access token. PKCE prevents stolen codes from being redeemed.
</details>

<details><summary><b>2. Access token vs ID token?</b></summary>

The access token is for APIs (audience = API) and authorizes calls. The ID token is for the client, to know who signed in (audience = client). APIs must reject ID tokens.
</details>

<details><summary><b>3. What does an API validate on a JWT?</b></summary>

The signature (using the IdP's JWKS keys), the issuer, the audience, the expiry and not-before times, and optionally the required scopes or claims. JWTs are only signed, not encrypted.
</details>

<details><summary><b>4. How do you prevent IDOR?</b></summary>

Resource-based authorization checks on every ID-based access (an owner or tenant check), queries scoped to the current user or tenant, 404 rather than 403 when existence is sensitive, and automated negative tests.
</details>

<details><summary><b>5. How do you revoke a JWT?</b></summary>

You can't easily, because it's self-contained. Use short lifetimes and revoke the refresh token at the IdP. For high-risk systems, add token introspection or a deny-list, at the cost of a lookup per request.
</details>

<details><summary><b>6. Why the BFF pattern for SPAs?</b></summary>

It keeps tokens out of the browser (where XSS can steal them). The browser only holds a secure, HttpOnly, SameSite cookie for the BFF, which holds the tokens server-side and proxies API calls.
</details>

---

## Exercises

1. Run Keycloak in compose with an imported realm. Get a token with the auth-code + PKCE flow using a tool (Bruno or Postman support it), decode it and identify every claim.
2. Implement the policies and the fallback policy. Write `.http` requests for 401, 403 and 200.
3. Write a failing IDOR test (user B reads user A's order), then fix it two ways: with a resource-based handler and with a scoped query.
4. Implement client credentials for a `ReportingWorker` that calls the API, with a token-caching `DelegatingHandler`.
5. Draw the PKCE flow from memory on paper. Do it again next week.

## Self-check

- [ ] I can explain OAuth2 vs OIDC, the token types and the main flows.
- [ ] I configured JWT bearer against a real IdP and know every validation step.
- [ ] My API is secure by default (fallback policy) and has IDOR tests.
- [ ] I know when to use cookies, bearer tokens or a BFF.

## Further reading

- *OAuth 2.0 Simplified* (Aaron Parecki, free online) and *OAuth 2.0 Security Best Current Practice* (RFC 9700).
- Microsoft Learn: *Configure JWT bearer authentication*, *Policy-based authorization*, *Resource-based authorization*.
- The Keycloak docs: *Server administration guide*, sections on clients, roles and mappers.
- OWASP *Authorization Cheat Sheet* and *JSON Web Token Cheat Sheet*.
