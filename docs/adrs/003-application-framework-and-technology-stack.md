# 003. Application Framework and Technology Stack

## Status

Accepted

## Context

Following the decisions to use SQLite with Litestream for data storage (ADR-002) and single-primary replication (ADR-001), we need to define the application framework and technology stack for building Groundwork Commons. The platform serves 5-50 members in hyperlocal communities and must balance:
- Developer productivity and maintainability
- Mobile-first user experience (primary interaction method)
- Optional web access for desktop users
- Modularity (node operators can choose which components to deploy)
- Future extensibility for native mobile apps

### Problem Statement
We need to choose:
- .NET framework version and deployment model
- Web application architecture (API, frontend, hosting model)
- Client-side framework for web experience
- Code sharing strategy between components
- Deployment flexibility for different node operator needs

### Constraints
- Must work within .NET ecosystem
- Mobile app will be primary interface (future)
- Web experience should feel app-like (responsive, fast)
- Node operators should choose which components to deploy (API-only, Web-only, both)
- Shared business logic between web and API
- Small-scale deployment (5-50 users, single server)
- Support for future iOS and Android native apps

### User Interaction Patterns
By the time this reaches production (2026+):
- **Primary:** Mobile app (iOS/Android) consuming REST API
- **Secondary:** Web browser (desktop/mobile) for users without app
- **Operator tools:** Web-based admin interface for node operators

### Alternatives Considered

**Monolithic MVC Application**
- Pros: Simple deployment, server-rendered pages, proven pattern
- Cons: Poor mobile app support, difficult to build native apps later
- Cons: No API-first architecture for future extensibility
- Rejected: Doesn't align with mobile-first future

**Next.js/React SPA + .NET API**
- Pros: Modern frontend experience, excellent mobile web UX
- Cons: Requires Node.js tooling alongside .NET
- Cons: Separate deployment complexity
- Cons: Different language/ecosystem from backend
- Rejected: Adds tooling complexity, splits ecosystem

**Blazor WebAssembly (WASM) + .NET API**
- Pros: Full .NET stack, code sharing, good offline support
- Cons: Large initial download size (~2-3MB)
- Cons: Slower initial load on mobile networks
- Cons: Requires separate API deployment anyway
- Rejected: Performance trade-offs not ideal for mobile-first

**ASP.NET Core Web API + Blazor Server (Chosen Approach)**
- Pros: .NET unified stack, fast initial load, small payload
- Pros: Real-time updates via SignalR (built-in)
- Pros: Modular deployment (API-only, Web-only, or both)
- Pros: Code sharing via .Core class library
- Pros: Future-proof for native mobile apps (consume same API)
- Cons: Requires persistent WebSocket connection (acceptable for local/neighborhood deployment)
- Chosen: Best balance of performance, maintainability, and future extensibility

**.NET Version Selection**
- .NET 8 (LTS): Supported until November 2026
- .NET 9 (STS): Supported until May 2025
- .NET 10+ (Future LTS): Expected November 2025, LTS until 2028+
- Decision: Target latest .NET at production time (~.NET 10 in 2026)

## Decision

We will build Groundwork Commons as a **modular .NET application** with the following architecture:

### 1. Application Structure

```
Groundwork.Commons/
├── Groundwork.Core/              # Shared class library
│   ├── Features/                 # Contains features by domain
│   ├── Infrastructure/           # External infra, databases, 3rd party integrations
│   └── Common/                   # Shared core features
│
├── Groundwork.Api/               # ASP.NET Core Web API
│   ├── Endpoints/                # REST endpoints
│   ├── Common/                   # Common services
│   └── Middleware/               # Auth, logging, etc.
│
├── Groundwork.Web/               # Blazor Server application
│   ├── Components/               # Blazor components
│   ├── Pages/                    # Blazor pages
│   └── Shared/                   # Shared layouts, components
│
└── Groundwork.Mobile/            # Future: .NET MAUI? (iOS/Android)
    └── (Future implementation)
```

### 2. Core Technology Stack

**Framework:**
- **.NET 10+** (latest LTS at production time, currently targeting .NET 10 expected Nov 2025)
- **C# 13+** (latest language features)
- **ASP.NET Core** for web hosting

**Database:**
- **SQLite** with **EF Core SQLite provider** (per ADR-002)
- **Litestream** for replication (per ADR-001)

**Shared Library:**
- **Groundwork.Core** - .NET Standard 2.1 / .NET 10 class library
- Shared business logic, models, data access across API and Web

### 3. Component: Web API (Groundwork.Api)

**Purpose:** RESTful API for future mobile apps and programmatic access

**Technology:**
- **ASP.NET Core Web API** with minimal APIs or controller-based endpoints
- **REST** conventions (JSON request/response)
- **JWT authentication** for mobile clients (stateless)
- **Swagger/OpenAPI** for API documentation

**Endpoints (Examples):**
```
POST   /api/auth/login
GET    /api/posts
POST   /api/posts
GET    /api/proposals
POST   /api/proposals/{id}/vote
DELETE /api/posts/{id}
```

**Deployment:** Can run standalone (API-only node) or alongside Blazor Server

### 4. Component: Web Application (Groundwork.Web)

**Purpose:** Web-based interface for members and node operators

**Technology:**
- **Blazor Server** (not WebAssembly)
- **SignalR** for real-time communication (built into Blazor Server)
- **Cookie-based authentication** for web sessions
- **Server-side rendering** with interactive components

**Why Blazor Server:**
- Fast initial load (small payload, server-rendered)
- Real-time updates via SignalR (perfect for social feed)
- Full .NET stack (code sharing with Core and API)
- No large WASM download
- Persistent connection acceptable for local/neighborhood network
- Leverage client-side Blazor features where possible for interactivity

**Future Consideration:** Offline functionality with sync-back (Future ADR needed for offline-first patterns)

**Deployment:** Can run standalone (Web-only node) or alongside API

### 5. Component: Mobile Apps (Future)

**Purpose:** Native iOS and Android applications

**Technology (Planned):**
- **.NET MAUI** (Multi-platform App UI)
- Consumes **Groundwork.Api** REST endpoints
- Shares **Groundwork.Core** models and business logic
- **JWT authentication** for API access

**Timeline:** Post-MVP, after web experience is proven

### 6. Authentication & Session Management

**Web (Blazor Server):**
- **Cookie-based authentication** via ASP.NET Core Identity
- Session state managed by ASP.NET Core (in-memory, not database)
- JWT token stored in cookie for API interoperability

**API (Mobile Apps):**
- **JWT tokens** issued via `/api/auth/login`
- Stateless authentication (token validated per request)
- Token refresh mechanism for long-lived sessions

**Credential Storage:**
- **SQLite database** with ASP.NET Core Identity tables (per ADR-002)
- **Password hashing:** ASP.NET Core Identity default (PBKDF2)
- Multi-device support: Each device gets own token/session

### 7. Deployment Modularity

Node operators can choose deployment configuration:

**Option 1: Full Stack (API + Web)**
- Run both Groundwork.Api and Groundwork.Web on same server
- Share same SQLite database
- Support web and future mobile users

**Option 2: Web Only**
- Run Groundwork.Web standalone
- Blazor Server handles all interactions
- No API exposed

**Option 3: API Only**
- Run Groundwork.Api standalone
- Mobile-only neighborhood (no web interface)
- Minimal resource usage

**Implementation:** Use environment variables or configuration to enable/disable components

### 8. Development and Build

**Build System:**
- **.NET CLI** (`dotnet build`, `dotnet publish`)
- **Self-contained deployment** (include .NET runtime in executable)
- Single executable per component for easy distribution

**Development Tools:**
- **Visual Studio 2025** or **Visual Studio Code** with C# Dev Kit
- **Entity Framework Core CLI** for migrations
- **Docker** (optional, for containerized deployment)

**Package Management:**
- **NuGet** for dependencies
- Minimal external dependencies (prioritize built-in .NET libraries)

## Consequences

### Positive Consequences

- **Unified .NET ecosystem:** Single language and framework across all components
- **Code sharing:** Groundwork.Core enables sharing models, services, and business logic
- **Modular deployment:** Node operators choose what to run based on needs
- **Future-proof for mobile:** API-first design supports native apps later
- **Real-time updates:** SignalR built into Blazor Server enables live social feed
- **Fast web experience:** Blazor Server provides quick load times
- **Developer productivity:** .NET tooling, EF Core, ASP.NET Core Identity reduce boilerplate
- **Small deployment footprint:** Self-contained executables, single SQLite file
- **Latest .NET features:** Targeting .NET 10+ gives modern language and framework capabilities
- **Strong typing:** C# provides compile-time safety across entire stack

### Negative Consequences

- **Blazor Server connection requirement:** Requires persistent SignalR connection (acceptable for local network)
- **SignalR overhead:** WebSocket connections consume server resources (minimal at 5-50 users)
- **No offline-first (yet):** Blazor Server requires connection; offline support needs future ADR
- **.NET learning curve:** Contributors must know C# and .NET ecosystem
- **Mobile app delay:** Native apps come later (web-first approach)
- **Session affinity:** In multi-node scenarios (future), need sticky sessions for Blazor Server
- **Breaking changes risk:** Targeting latest .NET means potential breaking changes during development

### Neutral Consequences

- **ASP.NET Core Identity tables:** Adds ~10 tables to SQLite schema (manageable)
- **Multiple projects:** Solution has 3+ projects (Core, Api, Web, future Mobile)
- **Deployment coordination:** Operators must update multiple executables when upgrading
- **Testing strategy:** Need to test both API and Web components separately
- **.NET version upgrades:** Plan to upgrade to latest LTS every 2-3 years

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (SQLite replication informs deployment model)
- ADR-002: Database Technology Selection (SQLite + EF Core integration)
- ADR-004: Authentication and Authorization Model (implements auth decisions here)
- Future ADR: Offline-First Functionality and Sync Strategy (Blazor offline patterns)
- Future ADR: Mobile Application Architecture (.NET MAUI implementation details)

### Implementation Phases

**Phase 1: Core + Web (MVP)**
- Build Groundwork.Core with domain models and EF Core setup
- Build Groundwork.Web with Blazor Server
- Implement authentication with ASP.NET Core Identity
- Basic social features (posts, comments, members)

**Phase 2: API Layer**
- Build Groundwork.Api with REST endpoints
- JWT authentication for programmatic access
- OpenAPI/Swagger documentation

**Phase 3: Advanced Web Features**
- Real-time notifications via SignalR
- Proposal and voting system
- Moderation tools
- Admin interface

**Phase 4: Mobile Apps**
- .NET MAUI iOS app
- .NET MAUI Android app
- Consume Groundwork.Api

### Performance Considerations
- **Blazor Server connection limit:** ASP.NET Core supports thousands of concurrent SignalR connections; 5-50 users is negligible
- **SQLite concurrent reads:** Excellent read performance for small-scale social feed
- **Memory footprint:** Estimate 50-100MB per node for entire application
- **Bandwidth:** SignalR uses minimal bandwidth for real-time updates (~1-10 KB/s per user)

### Future Enhancements
- **Progressive Web App (PWA):** Add service worker for offline support
- **Blazor Hybrid:** Explore Blazor Hybrid for mobile if MAUI proves heavyweight
- **gRPC API:** Consider gRPC for high-performance mobile communication (instead of REST)
- **Distributed caching:** If multi-node becomes common, add Redis for session sharing