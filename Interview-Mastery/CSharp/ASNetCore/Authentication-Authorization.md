# Authentication & Authorization in ASP.NET Core

---

## Authentication vs Authorization

### Authentication — "Who are you?"

- Authentication is the process of **verifying the identity** of a user
- It answers the question: "Is this user really who they claim to be?"
- Common mechanisms: username/password, biometrics, tokens, certificates
- An authenticated user has a verified identity (e.g., `ClaimsPrincipal`)
- If authentication fails, the user is **rejected before any resource access**

### Authorization — "What can you do?"

- Authorization happens **after** authentication
- It determines what an **authenticated user is allowed to do**
- It answers the question: "Does this user have permission to access this resource?"
- If authorization fails, the user gets a **403 Forbidden** response

### Why They Are Separate Concerns

- Separation of concerns: identity verification is different from permission checking
- You can change authorization rules without touching authentication logic
- Different teams or services can handle each independently
- You can authenticate users via Google but authorize them with your own roles

### Order of Operations

- **Authentication always runs before Authorization**
- In ASP.NET Core pipeline: `UseAuthentication()` must come before `UseAuthorization()`
- If a user is not authenticated, `[Authorize]` returns **401 Unauthorized**
- If a user is authenticated but lacks permission, `[Authorize]` returns **403 Forbidden**

---

## JWT (JSON Web Token)

### What Is JWT?

- JWT stands for **JSON Web Token**
- It is a compact, URL-safe token format defined by **RFC 7519**
- Used to securely transmit information between two parties as a JSON object
- Tokens are self-contained: they carry all needed user information
- Commonly used for stateless authentication in APIs

### JWT Structure

A JWT consists of three parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwiaXNzIjoiYXBpLmV4YW1wbGUuY29tIiwicm9sZXMiOlsiQWRtaW4iXSwiZXhwIjoxNzAwMDAwMDAwfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

- The token has exactly **3 parts**: Header, Payload, Signature
- Each part is **Base64URL encoded**

### Header

- Contains metadata about the token
- Typically specifies the **signing algorithm** and **token type**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

- `alg`: The algorithm used to sign the token (HS256, RS256, etc.)
- `typ`: Declares this is a JWT token

### Payload (Claims)

- Contains the **claims** — statements about the user
- Claims are key-value pairs with predefined and custom fields

**Standard Claims (Registered Claims):**

| Claim | Meaning | Example |
|-------|---------|---------|
| `sub` | Subject (user ID) | `"1234567890"` |
| `iss` | Issuer (who created the token) | `"https://auth.example.com"` |
| `aud` | Audience (who should use it) | `"https://api.example.com"` |
| `exp` | Expiration time (Unix timestamp) | `1700000000` |
| `nbf` | Not before (token invalid before this) | `1699900000` |
| `iat` | Issued at time | `1699999000` |
| `jti` | JWT ID (unique token identifier) | `"abc123"` |

**Custom Claims:**

```json
{
  "sub": "user-42",
  "name": "John Doe",
  "email": "john@example.com",
  "roles": ["Admin", "Editor"],
  "department": "Engineering",
  "permissions": ["read", "write", "delete"]
}
```

- Custom claims can hold anything, but avoid sensitive data

### Signature

- Ensures the token was **not tampered with** during transit
- Created by encoding the header and payload, then signing with a secret key

```
HMAC-SHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

- **HMAC-SHA256**: Symmetric — same key signs and verifies (single server)
- **RSA/ECDSA**: Asymmetric — private key signs, public key verifies (multiple services)

### Why JWT Is Stateless

- The server **does not store session data** after issuing the token
- All information needed is **encoded inside the token itself**
- Any server with the signing key can validate the token independently
- No database lookups needed for authentication
- Enables **horizontal scaling** — any server can serve any request

### JWT Expiry and Refresh Tokens

- JWTs should have a **short lifetime** (5–15 minutes for access tokens)
- After expiry, the user must obtain a new token
- **Refresh tokens** are long-lived tokens used to get new access tokens
- Refresh tokens are stored **server-side** (database or cache)
- The refresh token flow:
  1. Access token expires
  2. Client sends refresh token to `/auth/refresh`
  3. Server validates the refresh token and issues a new access token
  4. Old refresh token is invalidated (rotation)

### What NOT to Store in JWT

- **Never store sensitive data**: passwords, SSNs, credit card numbers
- JWT payload is Base64-encoded, **not encrypted** — anyone can read it
- **Don't store permissions that change frequently**: a user may be downgraded, but their old JWT still says "Admin"
- **Don't store secrets or keys** in the payload
- Limit JWT payload to identity information and role assignments

### Validating JWT in ASP.NET Core

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(1),
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)
            )
        };
    });
```

- Always validate the **signature** to prevent token forgery
- Always validate **expiration** (`exp` claim)
- Set `ClockSkew` to a small value (default is 5 minutes, which is too generous)
- Validate **issuer** and **audience** to ensure the token is intended for your API

### Token Lifetime and Refresh Strategy

- **Access token**: Short-lived (5–15 minutes) — used for API calls
- **Refresh token**: Long-lived (7–30 days) — used to obtain new access tokens
- Implement **token rotation**: issue a new refresh token each time one is used
- Store refresh tokens in a **secure, server-side store** (database, Redis)
- Implement **revocation** for logout and security incidents
- Consider **absolute expiry** on refresh tokens (e.g., max 30 days regardless of activity)

---

## OAuth 2.0

### What Is OAuth 2.0?

- OAuth 2.0 is an **authorization framework**, not an authentication protocol
- It allows a user to **grant a third-party application limited access** to their resources
- It defines how a client obtains **delegated access** to protected resources
- OAuth 2.0 replaced OAuth 1.0 and is the industry standard

### OAuth Roles

- **Resource Owner**: The user who owns the data (e.g., you on Google Drive)
- **Client**: The application requesting access to the user's resources (e.g., a third-party app)
- **Authorization Server**: Issues access tokens after authenticating the user and getting consent (e.g., Google's auth server)
- **Resource Server**: Hosts the protected resources and validates access tokens (e.g., Google Drive API)

### OAuth Flows (Grant Types)

#### Authorization Code Flow (Most Common)

- Best for **server-side web applications**
- The client gets an authorization code, then exchanges it for tokens
- Steps:
  1. User clicks "Login with Google"
  2. Browser redirects to Google's authorization endpoint
  3. User authenticates and consents
  4. Google redirects back to the client with an **authorization code**
  5. Client exchanges the code for an **access token** (server-side, no browser involved)
  6. Client uses the access token to call the resource server

#### Client Credentials Flow (Machine-to-Machine)

- Used when the client is **itself the resource owner** (no user involved)
- Common for backend services, daemons, microservices
- The client directly requests a token using its own credentials (client_id + client_secret)

```http
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=service-app-01
&client_secret=super-secret-value
```

#### Implicit Flow (Deprecated)

- Was used for **browser-based SPA applications**
- Access token returned directly in the URL fragment
- **Deprecated** due to security concerns (token exposed in browser history, referrer headers)
- Use **Authorization Code with PKCE** instead for SPAs

#### Password Grant (Deprecated)

- The client collects the user's username and password directly
- **Deprecated** — violates OAuth's core principle of delegated access
- Only used in legacy systems or highly trusted first-party apps

### Access Token vs Refresh Token

- **Access token**: Short-lived, used to access protected resources
- **Refresh token**: Long-lived, used to get new access tokens when the current one expires
- Access tokens are sent in the `Authorization` header: `Bearer <token>`
- Refresh tokens are **never** sent to resource servers
- Refresh tokens should be stored **securely** (HttpOnly cookie or server-side storage)

### OpenID Connect (OIDC)

- OpenID Connect is an **authentication layer** built on top of OAuth 2.0
- While OAuth 2.0 only handles **authorization**, OIDC handles **authentication**
- OIDC introduces the **ID Token** — a JWT containing user identity information
- Common claims in ID Token: `sub`, `name`, `email`, `picture`, `email_verified`
- Most OAuth providers (Google, Microsoft, Apple) support OIDC out of the box
- Use OIDC when you need to **log users in**, not just access their resources

---

## Cookie Authentication

### How Cookie Authentication Works

- User logs in with credentials
- Server creates an **encrypted cookie** containing user information
- The cookie is sent to the browser and stored automatically
- On subsequent requests, the browser **sends the cookie with each request**
- Server decrypts the cookie and reconstructs the user's identity

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/Account/Login";
        options.LogoutPath = "/Account/Logout";
        options.ExpireTimeSpan = TimeSpan.FromHours(8);
        options.SlidingExpiration = true;
    });
```

### ClaimsPrincipal Stored in Cookie

- The cookie does **not** store the user directly
- It stores a **serialized ClaimsPrincipal** (all claims about the user)
- The data is protected using **Data Protection API** (DPAPI)
- The cookie value is encrypted, signed, and optionally compressed

### Ticket-Based Encryption

- ASP.NET Core uses the **ticket format** for cookie authentication
- The "ticket" is a serialized `AuthenticationTicket` object
- The cookie value contains: `ProtectedData + "." + Signature`
- The signature ensures the cookie has **not been tampered with**
- The Data Protection system handles encryption/decryption automatically

### When to Use Cookie vs JWT

| Factor | Cookie | JWT |
|--------|--------|-----|
| **Type of app** | Server-rendered (MVC, Razor) | SPA, Mobile, API |
| **State** | Stateful (cookie sent automatically) | Stateless (manual header) |
| **XSS risk** | Lower (HttpOnly prevents JS access) | Higher (if stored in localStorage) |
| **CSRF risk** | Higher (cookies auto-sent) | Lower (no auto-send) |
| **Cross-domain** | Complicated (SameSite, CORS) | Easy (no cookie restrictions) |
| **Scalability** | Session affinity or shared store | Horizontally scalable |
| **Mobile support** | Limited | Excellent |

### SameSite Cookie Attribute

- `SameSite` controls when cookies are sent with cross-site requests
- **Strict**: Cookie only sent for same-site requests (most secure)
- **Lax**: Cookie sent for top-level navigations (GET requests from other sites)
- **None**: Cookie sent with all requests (requires `Secure` flag)
- ASP.NET Core defaults to `SameSite.Lax` for authentication cookies
- Set to `None` only when you need cross-site cookie usage (and always with `Secure`)

---

## ASP.NET Core Authentication

### Authentication Middleware

```csharp
var app = builder.Build();

// Authentication MUST come before Authorization
app.UseAuthentication();
app.UseAuthorization();
```

- `UseAuthentication()` adds the authentication middleware to the pipeline
- The middleware reads the token/cookie and populates `HttpContext.User`
- If the middleware is missing, `[Authorize]` will **never authenticate** anyone

### AddAuthentication() and AddJwtBearer()

```csharp
// Register authentication services
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = "https://auth.example.com",
        ValidateAudience = true,
        ValidAudience = "https://api.example.com",
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey))
    };
})
.AddCookie(CookieAuthenticationDefaults.AuthenticationScheme, options =>
{
    options.LoginPath = "/Account/Login";
});
```

- You can register **multiple authentication schemes** simultaneously
- The `DefaultAuthenticateScheme` determines which scheme is used by default
- Different endpoints can use different schemes via `[Authorize(AuthenticationSchemes = "...")]`

### AuthenticationSchemes

- Each authentication method registers with a **scheme name**
- `JwtBearerDefaults.AuthenticationScheme` = `"Bearer"`
- `CookieAuthenticationDefaults.AuthenticationScheme` = `"Cookies"`
- You can create custom schemes for specific requirements
- Useful when you need to support **multiple authentication methods** simultaneously

### [Authorize] Attribute

```csharp
[Authorize]
public IActionResult Dashboard()
{
    return View();
}
```

- Marks an action or controller as requiring authentication
- If user is not authenticated, returns **401 Unauthorized**
- Can be applied to a single action, a controller, or globally

### [AllowAnonymous] Attribute

```csharp
[Authorize]
public class HomeController : Controller
{
    public IActionResult Dashboard() { /* requires auth */ }

    [AllowAnonymous]
    public IActionResult PublicPage() { /* anyone can access */ }
}
```

- Overrides `[Authorize]` for specific actions
- Allows unauthenticated access to specific endpoints within a secured controller
- Most commonly used on login, register, and public pages

### Claims, ClaimsIdentity, ClaimsPrincipal

- **Claim**: A single name-value pair (e.g., `role = "Admin"`)
- **ClaimsIdentity**: A collection of claims for a single authentication scheme
- **ClaimsPrincipal**: One or more ClaimsIdentities (can have multiple identities from different auth schemes)

```
ClaimsPrincipal
├── ClaimsIdentity ("Bearer")
│   ├── Claim("sub", "user-42")
│   ├── Claim("email", "john@example.com")
│   └── Claim("role", "Admin")
└── ClaimsIdentity ("Cookies")
    ├── Claim("name", "John")
    └── Claim("department", "Engineering")
```

### HttpContext.User

- `HttpContext.User` returns the `ClaimsPrincipal` for the current request
- After authentication middleware runs, all claims are available here
- Used to read user information, check roles, and make authorization decisions

```csharp
[HttpGet("profile")]
public IActionResult GetProfile()
{
    var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var email = HttpContext.User.FindFirst(ClaimTypes.Email)?.Value;
    var isAdmin = HttpContext.User.IsInRole("Admin");

    return Ok(new { userId, email, isAdmin });
}
```

### Reading Claims from JWT

```csharp
[HttpGet("claims")]
public IActionResult GetClaims()
{
    // Access all claims
    var allClaims = HttpContext.User.Claims.ToList();

    // Get specific claim by type
    var sub = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var email = HttpContext.User.FindFirst("email")?.Value;
    var roles = HttpContext.User.FindAll(ClaimTypes.Role).Select(c => c.Value);

    return Ok(new { sub, email, roles });
}
```

- In ASP.NET Core with JWT, the `role` claim in the token is mapped to `ClaimTypes.Role`
- Custom claims are accessible by their claim type string
- `FindFirst()` returns the first matching claim or null
- `FindAll()` returns all claims of that type

---

## Authorization

### Role-Based Authorization

```csharp
// Single role
[Authorize(Roles = "Admin")]
public IActionResult AdminDashboard() { }

// Multiple roles (user must have at least one)
[Authorize(Roles = "Admin,Manager")]
public IActionResult ManagementDashboard() { }
```

- Simplest form of authorization
- User must have **at least one** of the specified roles
- Roles are typically stored in the `role` claim of the JWT or cookie

### Policy-Based Authorization

```csharp
// Define policies in Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanManageUsers", policy =>
        policy.RequireRole("Admin")
              .RequireClaim("department", "HR"));

    options.AddPolicy("MinimumAge", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));

    options.AddPolicy("PremiumUser", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(c => c.Type == "subscription" && c.Value == "premium")));
});

// Use policies in controllers
[Authorize(Policy = "CanManageUsers")]
public IActionResult ManageUsers() { }
```

- More flexible than role-based authorization
- Can combine multiple requirements
- Policies are reusable across the application

### Claim-Based Authorization

```csharp
// Require a specific claim
[Authorize(ClaimPermission = "CanWrite")]
public IActionResult WriteData() { }

// Or via policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanWrite", policy =>
        policy.RequireClaim("permission", "write"));
});
```

- Authorization based on specific claims the user possesses
- Claims can represent permissions, departments, locations, etc.
- More granular than roles alone

### Resource-Based Authorization

- Authorization decisions depend on the **resource being accessed**
- Example: "A user can only edit their own posts"
- Requires an `IAuthorizationHandler` that inspects the resource

```csharp
public class PostAuthorizationHandler : AuthorizationHandler<EditPostRequirement, Post>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        EditPostRequirement requirement,
        Post resource)
    {
        if (context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value == resource.AuthorId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

### Defining Policies in Program.cs

```csharp
builder.Services.AddAuthorization(options =>
{
    // Default policy applied when [Authorize] has no policy specified
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Fallback policy applied when no policy matches and no default policy is set
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Require a specific claim
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    // Combine multiple requirements
    options.AddPolicy("FullAccess", policy =>
        policy.RequireRole("Admin")
              .RequireClaim("verified", "true")
              .RequireAssertion(ctx => ctx.User.Identity?.IsAuthenticated == true));
});
```

### Authorization Handlers (IAuthorizationHandler)

- `IAuthorizationHandler` is responsible for **evaluating authorization requirements**
- It receives the `AuthorizationHandlerContext` and the requirement
- It calls `context.Succeed(requirement)` or does nothing (fail)

```csharp
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var birthDateClaim = context.User.FindFirst(c => c.Type == "birth_date");

        if (birthDateClaim == null)
            return Task.CompletedTask; // No claim = fail (don't succeed)

        var birthDate = DateTime.Parse(birthDateClaim.Value);
        var age = DateTime.Today.Year - birthDate.Year;

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

### Authorization Requirements (IAuthorizationRequirement)

- A requirement is a **data class** that holds authorization parameters
- It has no logic — it just defines what is required

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}
```

### Fallback Policies vs Default Policies

- **Default Policy**: Applied when `[Authorize]` is used **without specifying a policy**
  - If not set, unauthenticated users can access endpoints without `[Authorize]`
  - Set `options.DefaultPolicy` to require authentication globally

- **Fallback Policy**: Applied when the selected policy **does not exist**
  - Acts as a safety net to prevent unprotected access
  - Useful in large applications where you may forget to define a policy

```csharp
options.DefaultPolicy = new AuthorizationPolicyBuilder()
    .RequireAuthenticatedUser()
    .Build();

options.FallbackPolicy = new AuthorizationPolicyBuilder()
    .RequireAuthenticatedUser()
    .Build();
```

---

## RBAC (Role-Based Access Control)

### What Is RBAC?

- RBAC assigns **permissions to roles**, and **roles to users**
- Users never have direct permissions — they inherit them through roles
- Simplifies access control: instead of checking per-user, check per-role
- Industry standard for enterprise applications

### Why RBAC Is Preferred Over Hardcoded Checks

- **Maintainable**: Changing a role's permissions updates all users at once
- **Scalable**: Works well from small apps to enterprise systems
- **Auditable**: Easy to see who has what access
- **Separation of concerns**: Developers define roles; admins manage assignments
- **No code changes**: Adding a new role is a configuration change, not a deployment

### Dynamic RBAC

- Roles and permissions are **managed at runtime** via database or admin panel
- Users can be assigned/revoked roles without redeployment
- Permissions can be added to roles dynamically
- Requires a database to store role-permission mappings

### Database Schema for RBAC

```sql
-- Users table
CREATE TABLE Users (
    Id INT PRIMARY KEY IDENTITY,
    Username NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) NOT NULL
);

-- Roles table
CREATE TABLE Roles (
    Id INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(50) NOT NULL UNIQUE
);

-- Permissions table
CREATE TABLE Permissions (
    Id INT PRIMARY KEY IDENTITY,
    Name NVARCHAR(100) NOT NULL UNIQUE,
    Description NVARCHAR(500)
);

-- User-Role junction table (many-to-many)
CREATE TABLE UserRoles (
    UserId INT NOT NULL,
    RoleId INT NOT NULL,
    PRIMARY KEY (UserId, RoleId),
    FOREIGN KEY (UserId) REFERENCES Users(Id),
    FOREIGN KEY (RoleId) REFERENCES Roles(Id)
);

-- Role-Permission junction table (many-to-many)
CREATE TABLE RolePermissions (
    RoleId INT NOT NULL,
    PermissionId INT NOT NULL,
    PRIMARY KEY (RoleId, PermissionId),
    FOREIGN KEY (RoleId) REFERENCES Roles(Id),
    FOREIGN KEY (PermissionId) REFERENCES Permissions(Id)
);
```

### Implementing RBAC in ASP.NET Core

```csharp
// During login, build claims with roles and permissions
var userRoles = await _userManager.GetRolesAsync(user);
var userPermissions = await GetPermissionsForUserAsync(user.Id);

var claims = new List<Claim>
{
    new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
    new Claim(ClaimTypes.Email, user.Email!)
};

foreach (var role in userRoles)
    claims.Add(new Claim(ClaimTypes.Role, role));

foreach (var permission in userPermissions)
    claims.Add(new Claim("permission", permission));

var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
var principal = new ClaimsPrincipal(identity);

await HttpContext.SignInAsync(principal);
```

```csharp
// Use permission-based claims in authorization
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanDeleteUsers", policy =>
        policy.RequireClaim("permission", "users.delete"));

    options.AddPolicy("CanEditPosts", policy =>
        policy.RequireClaim("permission", "posts.edit"));
});

[Authorize(Policy = "CanDeleteUsers")]
public IActionResult DeleteUser(int id) { }
```

---

## LDAP/SSO Integration

### What Is LDAP?

- LDAP stands for **Lightweight Directory Access Protocol**
- It is a protocol for accessing and maintaining **directory information services**
- Commonly used with **Microsoft Active Directory** (AD)
- Stores hierarchical data: users, groups, organizational units
- Queries use distinguished names (DN) and filters

### What Is SSO?

- SSO stands for **Single Sign-On**
- A user authenticates **once** and gains access to **multiple applications**
- Eliminates the need to log in separately to each service
- Common SSO protocols: **SAML 2.0**, **OpenID Connect**, **Kerberos**
- Enterprise environments use LDAP + SSO extensively

### How LDAP-Based SSO Works

1. User tries to access Application A
2. Application A redirects to the **SSO server** (LDAP-backed)
3. User authenticates with LDAP credentials (if not already logged in)
4. SSO server issues a **token/ticket**
5. Application A validates the ticket and creates a local session
6. User accesses Application B — SSO server recognizes them, no re-login needed

### Novell.Directory.Ldap.NETStandard

```csharp
// Install NuGet package
// Install-Package Novell.Directory.Ldap.NETStandard

using Novell.Directory.Ldap;

public class LdapService
{
    private readonly string _ldapHost = "ldap.company.com";
    private readonly int _ldapPort = 389;
    private readonly string _baseDn = "dc=company,dc=com";

    public bool Authenticate(string username, string password)
    {
        using var connection = new LdapConnection();
        connection.Connect(_ldapHost, _ldapPort);
        connection.Bind($"cn={username},{_baseDn}", password);
        return connection.Bound;
    }

    public List<string> GetUserGroups(string username)
    {
        using var connection = new LdapConnection();
        connection.Connect(_ldapHost, _ldapPort);
        connection.Bind("cn=admin,dc=company,dc=com", "admin-password");

        var searchFilter = $"(&(objectClass=user)(cn={username}))";
        var results = connection.Search(
            _baseDn,
            LdapConnection.ScopeSub,
            searchFilter,
            new[] { "memberOf" },
            false
        );

        var groups = new List<string>();
        while (results.HasMore())
        {
            var entry = results.Next();
            var memberOf = entry.GetAttributeValues("memberOf");
            if (memberOf != null)
            {
                while (memberOf.MoveNext())
                    groups.Add(memberOf.Current!.ToString()!);
            }
        }
        return groups;
    }
}
```

### Integrating with Active Directory

- Use **WindowsIdentity** for Windows-authenticated environments
- Use **System.DirectoryServices.AccountManagement** for AD queries
- Map AD groups to application roles during login

```csharp
using System.DirectoryServices.AccountManagement;

public class AdService
{
    public bool ValidateCredentials(string username, string password)
    {
        using var context = new PrincipalContext(
            ContextType.Domain,
            "COMPANY",
            username,
            password
        );
        return context.ValidateCredentials(username, password);
    }

    public List<string> GetAdGroups(string username)
    {
        using var context = new PrincipalContext(ContextType.Domain, "COMPANY");
        using var user = UserPrincipal.FindByIdentity(context, username);

        if (user == null) return new List<string>();

        var groups = user.GetAuthorizationGroups();
        return groups.OfType<GroupPrincipal>()
                     .Select(g => g.Name)
                     .ToList();
    }
}
```

### Mapping LDAP Groups to Application Roles

```csharp
public class LdapRoleMapping
{
    private readonly Dictionary<string, string> _groupToRoleMap = new()
    {
        { "CN=IT-Admins,OU=Groups,DC=company,DC=com", "Admin" },
        { "CN=Developers,OU=Groups,DC=company,DC=com", "Developer" },
        { "CN=Support-Team,OU=Groups,DC=company,DC=com", "Support" }
    };

    public List<string> MapGroupsToRoles(List<string> ldapGroups)
    {
        var roles = new List<string>();
        foreach (var group in ldapGroups)
        {
            if (_groupToRoleMap.TryGetValue(group, out var role))
                roles.Add(role);
        }
        return roles;
    }
}
```

---

## Common Mistakes

### Storing Secrets in JWT Payload

- JWT payload is **Base64-encoded, not encrypted**
- Anyone can decode and read the payload by pasting it into jwt.io
- **Never store**: passwords, API keys, connection strings, PII (unless required)
- Store only non-sensitive identity claims (user ID, email, roles)

### Not Validating JWT Signature

- If you don't validate the signature, an attacker can **forge any token**
- Always set `ValidateIssuerSigningKey = true`
- Use a strong, randomly generated key (at least 256 bits for HMAC)
- Never disable signature validation, even in development

### Using Wrong HttpOnly Cookie Settings

- `HttpOnly`: Must be `true` to prevent JavaScript access (XSS protection)
- `Secure`: Must be `true` in production (HTTPS only)
- `SameSite`: Use `Lax` for most apps, `Strict` for banking-sensitive apps
- Forgetting `Secure` flag exposes cookies to interception over HTTP

### Hardcoding Authorization Rules

```csharp
// BAD: hardcoded role check scattered everywhere
if (user.Role == "Admin") { /* do something */ }

// GOOD: use policy-based authorization
[Authorize(Policy = "CanManageUsers")]
public IActionResult ManageUsers() { }
```

- Hardcoding makes authorization **difficult to audit and maintain**
- Changes require code modifications and redeployment
- Use policies and RBAC to centralize authorization logic

### Not Refreshing Expired Tokens

- Clients must handle **401 responses** by attempting a token refresh
- Implement exponential backoff for failed refresh attempts
- Store refresh tokens securely (HttpOnly cookie, not localStorage)
- Implement a **token refresh interceptor** in HTTP clients

### Exposing JWT in URLs

- JWTs should be sent in the `Authorization` header, **not in query strings**
- Tokens in URLs get logged in server logs, browser history, and referrer headers
- Always use: `Authorization: Bearer <token>`
- Never use: `?token=<jwt>` for authentication

---

## Interview Questions

### Q1: What is the difference between authentication and authorization?

**Answer:** Authentication verifies **who the user is** (identity), while authorization determines **what the user can do** (permissions). Authentication always runs first. You authenticate with credentials (password, token, biometrics), and once identified, authorization checks your roles/permissions against the requested resource.

---

### Q2: What are the three parts of a JWT?

**Answer:** JWT has three parts separated by dots:
- **Header**: Contains the algorithm (alg) and token type (typ)
- **Payload**: Contains claims (sub, iss, exp, roles, custom data)
- **Signature**: Ensures the token wasn't tampered with, created by signing header + payload with a secret key

---

### Q3: Why is JWT considered stateless?

**Answer:** JWT is stateless because all user information is encoded **inside the token itself**. The server doesn't store session data after issuing the token. Any server with the signing key can validate the token independently, enabling horizontal scaling without shared session storage.

---

### Q4: What is the difference between an access token and a refresh token?

**Answer:** An **access token** is short-lived (5–15 minutes) and used to access protected resources. A **refresh token** is long-lived (days to weeks) and used to obtain new access tokens when the current one expires. Refresh tokens are stored server-side and can be revoked, while access tokens are stateless.

---

### Q5: Explain the OAuth 2.0 Authorization Code flow.

**Answer:**
1. User clicks "Login with Provider"
2. Browser redirects to the authorization server
3. User authenticates and grants consent
4. Authorization server redirects back with an **authorization code**
5. Client exchanges the code for tokens (server-side HTTP call)
6. Client uses the access token to call the resource server

The code exchange happens server-to-server, so the access token is never exposed to the browser.

---

### Q6: What is the Client Credentials flow used for?

**Answer:** The Client Credentials flow is used for **machine-to-machine** communication where no user is involved. The client authenticates directly with the authorization server using its client_id and client_secret, and receives an access token. Common for backend services, daemons, and microservice-to-microservice calls.

---

### Q7: When should you use cookie authentication vs JWT?

**Answer:** Use **cookies** for server-rendered MVC/Razor applications where the browser automatically sends cookies. Use **JWT** for APIs, SPAs, and mobile applications where you need stateless authentication and cross-domain support. Cookies are simpler for traditional web apps; JWTs are better for distributed systems.

---

### Q8: What is the SameSite cookie attribute?

**Answer:** SameSite controls when cookies are sent with cross-site requests. `Strict` only sends cookies for same-site requests. `Lax` sends cookies for top-level navigations (default in ASP.NET Core). `None` sends cookies with all requests (requires Secure flag). SameSite prevents CSRF attacks by limiting when cookies are sent.

---

### Q9: What is OpenID Connect and how does it differ from OAuth 2.0?

**Answer:** OAuth 2.0 is an **authorization** framework that grants limited access to resources. OpenID Connect is an **authentication** layer built on top of OAuth 2.0. OIDC introduces an ID Token that contains user identity information (name, email, etc.). Use OAuth for authorization, OIDC when you need to authenticate users.

---

### Q10: What is role-based authorization and what are its limitations?

**Answer:** Role-based authorization checks if a user has specific roles: `[Authorize(Roles = "Admin")]`. Limitations include: coarse granularity (all admins have same permissions), role explosion (too many roles for fine-grained needs), and difficulty handling conditional access (e.g., "only own posts"). Policy-based authorization addresses these limitations.

---

### Q11: How do you implement policy-based authorization in ASP.NET Core?

**Answer:** Define policies in `Program.cs` using `AddAuthorization()`:

```csharp
options.AddPolicy("CanManageUsers", policy =>
    policy.RequireRole("Admin")
          .RequireClaim("department", "HR"));
```

Then apply with `[Authorize(Policy = "CanManageUsers")]`. Policies can combine role, claim, and custom requirement checks.

---

### Q12: What is an IAuthorizationHandler?

**Answer:** An `IAuthorizationHandler` evaluates authorization requirements at runtime. It receives the `AuthorizationHandlerContext` and the requirement, then calls `context.Succeed(requirement)` if authorized. It's used for resource-based authorization where decisions depend on the actual resource being accessed (e.g., "user can only edit their own posts").

---

### Q13: Explain RBAC and its database schema.

**Answer:** RBAC (Role-Based Access Control) assigns permissions to roles, and roles to users. The schema has four tables: **Users**, **Roles**, **Permissions**, **UserRoles** (junction), and **RolePermissions** (junction). Users never have direct permissions — they inherit them through roles. This makes access control maintainable, auditable, and scalable.

---

### Q14: How does LDAP-based SSO work?

**Answer:** LDAP-based SSO uses a directory service (like Active Directory) as the central authentication store. When a user logs in, the application queries LDAP to verify credentials. An SSO server issues a token/ticket after authentication. Other applications trust this ticket, so the user doesn't need to re-authenticate. This enables single sign-on across multiple applications.

---

### Q15: What should you never store in a JWT?

**Answer:** Never store sensitive data like passwords, API keys, connection strings, or PII (unless encrypted separately). JWT payload is Base64-encoded, not encrypted — anyone can decode it. Also, avoid storing permissions that change frequently, as the JWT remains valid until expiry. Store such data server-side and reference by user ID.

---

### Q16: How do you validate a JWT in ASP.NET Core?

**Answer:** Use `AddJwtBearer()` with `TokenValidationParameters`:
- `ValidateIssuer = true` with `ValidIssuer`
- `ValidateAudience = true` with `ValidAudience`
- `ValidateLifetime = true` with small `ClockSkew`
- `ValidateIssuerSigningKey = true` with the correct signing key
- Never disable signature validation, even in development

---

### Q17: What is the difference between 401 and 403 responses?

**Answer:** **401 Unauthorized** means the user is **not authenticated** (missing or invalid credentials). **403 Forbidden** means the user is **authenticated but not authorized** (lacks permission). In ASP.NET Core: `[Authorize]` returns 401 if the user isn't authenticated, 403 if they don't have the required role/policy.

---

### Q18: How do you handle JWT token refresh?

**Answer:** Implement a refresh token flow:
1. Client stores access token and refresh token securely
2. When access token expires, client calls `/auth/refresh` with the refresh token
3. Server validates the refresh token, issues a new access token, and optionally a new refresh token (rotation)
4. Implement token rotation to prevent refresh token reuse
5. Handle failed refresh attempts (redirect to login)

---

### Q19: What is the difference between DefaultPolicy and FallbackPolicy?

**Answer:** **DefaultPolicy** is applied when `[Authorize]` is used without specifying a policy — it defines what "unauthenticated users are rejected" means. **FallbackPolicy** is applied when the selected policy doesn't exist — it's a safety net for undefined policies. Both should typically require authentication to prevent accidental unprotected endpoints.

---

### Q20: How do you read claims from the current user in ASP.NET Core?

**Answer:**
```csharp
var userId = HttpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var email = HttpContext.User.FindFirst("email")?.Value;
var roles = HttpContext.User.FindAll(ClaimTypes.Role).Select(c => c.Value);
var isAdmin = HttpContext.User.IsInRole("Admin");
```

`FindFirst()` returns the first matching claim, `FindAll()` returns all claims of a type. Custom claims are accessed by their claim type string.

---

### Q21: What is token rotation and why is it important?

**Answer:** Token rotation means issuing a **new refresh token** every time one is used to obtain a new access token. The old refresh token is invalidated. This limits the window of exposure if a refresh token is compromised. Combined with an absolute expiry on refresh tokens, it provides strong protection against token theft.

---

### Q22: How would you implement dynamic RBAC in an ASP.NET Core application?

**Answer:**
1. Create database tables: Users, Roles, Permissions, UserRoles, RolePermissions
2. During login, query the database for the user's roles and all associated permissions
3. Add roles as `ClaimTypes.Role` claims and permissions as custom claims to the ClaimsPrincipal
4. Define policies that require specific permission claims
5. Apply `[Authorize(Policy = "CanDeleteUsers")]` to endpoints
6. Provide an admin UI to manage roles, permissions, and assignments at runtime
