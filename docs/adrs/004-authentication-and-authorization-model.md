# 004. Authentication and Authorization Model

## Status

Accepted

## Context

Groundwork Commons requires a security model that balances accessibility, security, and the unique characteristics of hyperlocal communities where members know each other personally and share physical proximity. The authentication and authorization system must work across:
- Web interface (Blazor Server with cookie-based auth)
- API (REST endpoints with JWT tokens for future mobile apps)
- Failover scenarios (credentials must survive primary node failure)
- Multiple devices per member
- Password recovery in communities where members know each other in person

### Problem Statement
We need to define:
- How member identities are created and stored
- Authentication mechanisms for web and API
- Session management strategies
- Password reset and account recovery flows
- Role-based authorization model
- How administrators can help members with account issues
- Multi-device access patterns

### Constraints
- Must integrate with SQLite database (ADR-002)
- Credentials must replicate with database (ADR-001) to survive failover
- Support both web (cookie) and mobile (JWT) authentication patterns (ADR-003)
- Communities are small (5-50 members) and members know each other
- Node operators are trusted community members with physical proximity
- Simple role model (admin, moderator, member)
- Must use proven security patterns (no custom crypto)

### Unique Community Characteristics
- **Physical proximity:** Members can verify identities in person
- **Trust relationships:** Admins personally know members
- **Disaster scenarios:** Normal email/SMS recovery may not work post-disaster
- **Low-tech users:** Some members may not be technically sophisticated
- **Community ownership:** No external identity providers required

### Alternatives Considered

**Custom Authentication System**
- Pros: Full control, minimal dependencies
- Cons: High risk of security vulnerabilities
- Cons: Reinventing well-tested patterns
- Rejected: Security-critical code should use proven libraries

**OAuth/OIDC Only (External Providers)**
- Pros: Delegates auth to Google, Microsoft, etc.
- Cons: Requires external internet connectivity
- Cons: Corporate dependency conflicts with community ownership
- Cons: Fails in isolated network scenarios
- Rejected: External dependencies unacceptable for resilience goals

**Passkeys/WebAuthn Only**
- Pros: Passwordless, phishing-resistant
- Cons: Device-dependent (lose device = lose access)
- Cons: Complexity for non-technical users
- Cons: Recovery mechanisms complex
- Rejected: Too cutting-edge for community scale; can add later

**ASP.NET Core Identity with SQLite (Chosen Approach)**
- Pros: Battle-tested, integrates with EF Core and SQLite
- Pros: Built-in password hashing (PBKDF2), secure defaults
- Cons: Supports username/password, email confirmation, password reset
- Pros: Extensible for future OAuth providers if desired
- Pros: Works with both cookie and JWT authentication
- Chosen: Best balance of security, simplicity, and .NET integration

## Decision

We will use **ASP.NET Core Identity** with **SQLite storage** for authentication and a **simple role-based authorization model**.

### 1. Identity Storage

**Technology:**
- **ASP.NET Core Identity** with Entity Framework Core SQLite provider
- Identity tables stored in same SQLite database as application data
- All credential data replicates via Litestream (per ADR-001)

**Database Schema:**
ASP.NET Core Identity creates the following tables:
```
AspNetUsers              # Member accounts
AspNetRoles              # Roles (Admin, Moderator, Member)
AspNetUserRoles          # User-to-role mappings
AspNetUserClaims         # Additional user claims
AspNetUserLogins         # External OAuth logins (optional)
AspNetUserTokens         # Password reset, email confirmation tokens
AspNetRoleClaims         # Role-based permissions (if needed)
```

**Custom User Model:**
```csharp
public class GroundworkUser : IdentityUser
{
    public string DisplayName { get; set; }
    public DateTime JoinedAt { get; set; }
    public string ProfileData { get; set; } // JSON for flexible fields
    public bool IsActive { get; set; }
}
```

### 2. Authentication Mechanisms

**Web (Blazor Server):**
- **Cookie-based authentication** via ASP.NET Core Identity
- Login form posts to `/Account/Login`
- Authentication cookie set with sliding expiration (14 days default)
- JWT token embedded in claims for API interoperability
- HttpOnly, Secure, SameSite=Strict cookie settings

**API (Future Mobile Apps):**
- **JWT bearer tokens** for stateless authentication
- Login endpoint: `POST /api/auth/login` returns JWT
- Token contains user claims (ID, username, roles)
- Configurable expiration (7-30 days for mobile)
- Refresh token support for long-lived sessions

**Shared Validation:**
- Both cookie and JWT validated against same user database
- User account changes (password reset, role change) reflected immediately
- Account lockout applies to both web and API sessions

### 3. Password Security

**Hashing:**
- **ASP.NET Core Identity default:** PBKDF2 with HMAC-SHA256
- 10,000 iterations (configurable, can increase)
- Per-user salt (handled automatically by Identity)

**Password Requirements (Configurable):**
```csharp
options.Password.RequireDigit = false;
options.Password.RequireLowercase = false;
options.Password.RequireUppercase = false;
options.Password.RequireNonAlphanumeric = false;
options.Password.RequiredLength = 8;
```

**Reasoning:** Balance security with usability for community scale. Can be tightened if needed.

### 4. Session Management

**Web Sessions:**
- **Cookie-based** with in-memory session state (not database)
- No persistent session storage (reduces database writes)
- Sliding expiration: Cookie refreshes on activity
- Multiple devices: Each device gets own cookie
- Session terminates on explicit logout or cookie expiration

**API Sessions (JWT):**
- **Stateless tokens:** No server-side session storage
- Token validated per request via signature verification
- Multiple devices: Each device gets own JWT token
- Token revocation: Store revoked tokens in database (if needed)
- Refresh token flow: Issue new JWT when near expiration

**Failover Behavior:**
- Web cookies survive failover (same encryption key across nodes)
- JWT tokens remain valid (same signing key across nodes)
- **Implementation:** Store cookie/JWT signing keys in SQLite database (replicated)

### 5. Account Recovery

Given physical proximity and personal relationships in neighborhoods:

**Password Reset - Standard Flow:**
1. Member clicks "Forgot Password"
2. Enters username or email
3. Receives password reset link via email
4. Clicks link, sets new password

**Password Reset - Admin-Assisted Flow (Unique to Groundwork):**
1. Member contacts admin in person or via alternate channel
2. Admin verifies identity in person (leverage physical proximity)
3. Admin uses privileged function: "Reset Password for User"
4. System generates temporary password or reset link
5. Admin provides to member via trusted channel
6. Member logs in and sets new password

**Post-Disaster Recovery:**
- If email infrastructure unavailable, admin-assisted reset is primary method
- Admin can verify identity in person (neighborhood context)
- Database replication ensures accounts survive node failures

**Account Lockout:**
- 5 failed login attempts triggers 15-minute lockout
- Admin can manually unlock accounts
- Prevents brute-force attacks while allowing admin intervention

### 6. Authorization Model

**Roles:**
- **Member** (default role for all users)
- **Moderator** (can delete posts, ban members, flag content)
- **Admin** (all moderator powers + member management + node configuration)

**Role Hierarchy:**
```
Admin > Moderator > Member
```

**Permissions by Role:**

| Action | Member | Moderator | Admin |
|--------|--------|-----------|-------|
| Create posts/comments | ✅ | ✅ | ✅ |
| Edit own posts | ✅ | ✅ | ✅ |
| Delete own posts | ✅ | ✅ | ✅ |
| Edit others' posts | ❌ | ❌ | ❌ |
| Delete others' posts | ❌ | ✅ | ✅ |
| Flag content (warning) | ❌ | ✅ | ✅ |
| Ban members | ❌ | ✅ | ✅ |
| Create proposals | ✅ | ✅ | ✅ |
| Vote on proposals | ✅ | ✅ | ✅ |
| Manage member accounts | ❌ | ❌ | ✅ |
| Assign roles | ❌ | ❌ | ✅ |
| Node configuration | ❌ | ❌ | ✅ |
| Password reset assistance | ❌ | ❌ | ✅ |

**Special Note on Editing:**
- Members can only edit their own content
- Moderators and admins CANNOT edit others' content (prevent abuse)
- Deletion and flagging are the moderation tools (preserves content integrity)

**Democratic Role Changes:**
- Special proposal types can automatically change roles based on vote outcome
- Example: "Remove Steve from Moderator" proposal passes → system removes Moderator role
- Implemented via proposal system (see ADR-005)

### 7. Member Onboarding

**Invite Code System:**
- Admins generate single-use or multi-use invite codes
- Invite code required to create account (prevents open registration)
- Invite codes stored in database with usage tracking

**Registration Flow:**
1. Member receives invite code from admin (in person, SMS, email, etc.)
2. Member navigates to `/register`
3. Enters invite code, username, email, password
4. System validates invite code, creates account
5. Member assigned "Member" role by default
6. Invite code marked as used (if single-use) or usage count incremented

**Invite Code Types:**
- **Single-use:** One account creation, then expires
- **Multi-use:** N accounts, then expires
- **Unlimited:** No usage limit (for open community growth)
- **Time-limited:** Expires after date

**Admin Management:**
- Admins can view active invite codes
- Admins can revoke unused invite codes
- Admins can see which members used which codes (audit trail)

### 8. Multi-Device Access

**Web Sessions:**
- Each device/browser gets independent cookie
- Member can be logged in on desktop, tablet, phone simultaneously
- Logout on one device doesn't affect others (local cookie removal)
- Global logout option: "Log out all devices" invalidates all cookies

**API Sessions (JWT):**
- Each mobile device gets independent JWT token
- Member can have iOS app, Android app, tablet logged in simultaneously
- Token refresh independent per device
- Revoke specific device: Store token ID in revocation list

**Security Consideration:**
- No limit on concurrent sessions (small community, trusted network)
- Can add session limits if abuse becomes issue

### 9. Security Configuration

**Cookie Settings:**
```csharp
services.ConfigureApplicationCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always; // HTTPS only
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.ExpireTimeSpan = TimeSpan.FromDays(14);
    options.SlidingExpiration = true;
    options.LoginPath = "/Account/Login";
    options.LogoutPath = "/Account/Logout";
});
```

**JWT Settings:**
```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "groundwork-commons",
            ValidAudience = "groundwork-app",
            IssuerSigningKey = new SymmetricSecurityKey(GetSigningKeyFromDatabase())
        };
    });
```

**Signing Key Management:**
- JWT signing key stored in SQLite database (replicated)
- Cookie encryption key stored in SQLite database (replicated)
- Keys persist across failover (same keys in replica database)
- Key rotation strategy: Manual rotation via admin tool (future enhancement)

## Consequences

### Positive Consequences

- **Battle-tested security:** ASP.NET Core Identity is used by millions of applications
- **Built-in password security:** PBKDF2 hashing with proper salting
- **Dual auth support:** Cookie for web, JWT for mobile, using same user store
- **Failover resilience:** Credentials and signing keys replicate with database
- **Multi-device friendly:** Independent sessions per device without conflict
- **Admin-assisted recovery:** Leverages physical proximity for password resets
- **Invite code control:** Prevents spam, allows community growth management
- **Simple role model:** Easy to understand and implement
- **Extensible:** Can add OAuth providers later without rewriting
- **No custom crypto:** Reduces risk of security vulnerabilities

### Negative Consequences

- **Admin trust requirement:** Admins can reset passwords (acceptable given neighborhood context)
- **Email dependency:** Standard password reset requires email infrastructure (mitigated by admin-assisted reset)
- **Token revocation complexity:** JWT stateless design makes global logout complex (can add revocation list)
- **Key management:** Must carefully backup signing keys in replicated database
- **Cookie/JWT expiration:** Members must re-login periodically (acceptable security trade-off)
- **No fine-grained permissions:** Simple role model may limit future flexibility (can extend with claims)
- **Identity table bloat:** ASP.NET Core Identity adds ~10 tables to schema (acceptable)

### Neutral Consequences

- **Password requirements:** Configurable, can be tightened if needed
- **Session limits:** No concurrent session limits initially (can add later)
- **OAuth support:** Not used initially, but framework supports adding later
- **Two-factor authentication:** Not implemented in MVP (can add via ASP.NET Core Identity)
- **Audit logging:** Need to implement login/logout tracking separately

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (credentials replicate with database)
- ADR-002: Database Technology Selection (ASP.NET Core Identity uses SQLite)
- ADR-003: Application Framework and Technology Stack (implements dual auth)
- ADR-005: Democratic Governance and Proposal System (role changes via proposals)

### Implementation Phases

**Phase 1: Basic Authentication (MVP)**
- ASP.NET Core Identity setup with SQLite
- Cookie-based web authentication
- Username/password login
- Simple registration (no invite codes yet)
- Admin, Moderator, Member roles

**Phase 2: Invite System**
- Invite code generation and validation
- Admin UI for managing invite codes
- Registration requires invite code

**Phase 3: API Authentication**
- JWT token issuance
- API endpoints for login
- Token refresh mechanism

**Phase 4: Advanced Recovery**
- Email-based password reset
- Admin-assisted password reset UI
- Account lockout management

**Phase 5: Security Enhancements**
- Token revocation list
- Global logout functionality
- Audit logging for security events
- Optional: Two-factor authentication

### Security Best Practices

1. **HTTPS Only:** Enforce HTTPS in production (ASP.NET Core Kestrel + reverse proxy)
2. **Secure Defaults:** Use ASP.NET Core Identity defaults unless specific reason to change
3. **Regular Updates:** Keep ASP.NET Core and dependencies updated for security patches
4. **Audit Logging:** Log all authentication events (login, logout, password change, role change)
5. **Rate Limiting:** Consider rate limiting login attempts (beyond account lockout)
6. **Data Protection:** ASP.NET Core Data Protection API handles cookie/token encryption

### References
- [ASP.NET Core Identity Documentation](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity)
- [JWT Authentication in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn)
- [Password Hashing in ASP.NET Core Identity](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [ASP.NET Core Data Protection](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/)

### Future Enhancements
- **Two-factor authentication (2FA):** SMS or authenticator app codes
- **OAuth/OIDC providers:** Optional Google, Microsoft, GitHub login
- **Passkeys/WebAuthn:** Passwordless authentication option
- **Biometric login:** For mobile apps (Face ID, Touch ID)
- **Magic links:** Email-based passwordless login
- **Session activity tracking:** See all active sessions, revoke specific devices