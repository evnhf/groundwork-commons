# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Groundwork Commons** is a platform for hyperlocal community infrastructure (5-50 people sharing physical proximity). Built on .NET with SQLite, it enables neighborhood-scale social coordination with democratic governance and data sovereignty.

**Current Status**: Concept Development (October 2025) - Planning documentation exists, implementation has not begun.

## Architecture Decisions

All major technical decisions are documented in [Architecture Decision Records](docs/adrs/README.md). Read these before making implementation choices:

- [ADR-001](docs/adrs/001-data-replication-strategy.md): Single-primary node with continuous replication via Litestream
- [ADR-002](docs/adrs/002-database-technology-selection.md): SQLite with JSON1 extension + Litestream for replication
- [ADR-003](docs/adrs/003-application-framework-and-technology-stack.md): .NET 10+ with modular architecture (API + Blazor Server)
- [ADR-004](docs/adrs/004-authentication-and-authorization-model.md): ASP.NET Core Identity with dual auth (cookies + JWT)
- [ADR-005](docs/adrs/005-democratic-governance-and-proposal-system.md): Flexible proposal system with automated enforcement
- [ADR-006](docs/adrs/006-core-data-model-and-schema-design.md): Unified polymorphic content model with type-specific metadata
- [ADR-007](docs/adrs/007-content-features-and-capabilities.md): Markdown content with flexible media storage (filesystem or S3-compatible)
- [ADR-008](docs/adrs/008-deployment-and-configuration-model.md): Docker Compose deployment with setup wizard and automatic migrations

## Planned Technology Stack

When implementation begins, this project will use:

- **.NET 10+** (latest LTS at production time)
- **SQLite** with JSON1 extension via EF Core
- **Litestream** for continuous database replication
- **ASP.NET Core Web API** for REST endpoints (future mobile apps)
- **Blazor Server** for web interface
- **ASP.NET Core Identity** for authentication

## Planned Project Structure

The implementation will follow this modular architecture:

```
Groundwork.Commons/
├── Groundwork.Core/              # Shared class library
│   ├── Features/                 # Domain features
│   ├── Infrastructure/           # Database, external integrations
│   └── Common/                   # Shared utilities
│
├── Groundwork.Api/               # ASP.NET Core Web API
│   ├── Endpoints/                # REST endpoints
│   ├── Common/                   # API services
│   └── Middleware/               # Auth, logging
│
├── Groundwork.Web/               # Blazor Server application
│   ├── Components/               # Blazor components
│   ├── Pages/                    # Blazor pages
│   └── Shared/                   # Layouts, shared components
│
└── Groundwork.Mobile/            # Future: .NET MAUI
```

## Key Design Principles

1. **Hyperlocal Scale**: Design for 5-50 users, not millions. Favor simplicity over complex distributed systems.

2. **Single-Primary Architecture**: One authoritative write node with continuous replication to backups. No multi-master complexity.

3. **Resilience Through Replication**: All data (including credentials, signing keys) must replicate via Litestream to survive node failover.

4. **Democratic Governance**: Special proposal types automatically enforce vote outcomes (role changes, bans) without admin override.

5. **Modular Deployment**: Node operators choose which components to run (API-only, Web-only, or both).

6. **Hybrid Data Model**: Use relational tables for structured data (members, roles, permissions), JSON columns for flexible content (posts, profiles).

7. **Community Ownership**: No external dependencies required. Must work in isolated networks.

## Database Design Approach

Per ADR-002, use this hybrid approach:

**Relational tables** for:
- Members, roles, permissions
- Audit logs, moderation actions
- Relationships requiring referential integrity

**JSON columns** for:
- Post content and metadata
- User profiles and custom fields
- Settings and configuration

## Authentication Implementation Notes

Per ADR-004:

- Use **ASP.NET Core Identity** with SQLite storage
- **Cookie-based auth** for Blazor Server web sessions
- **JWT tokens** for future mobile API access
- Store JWT/cookie signing keys in SQLite database (they must replicate)
- Implement **invite code system** to control community membership
- Support **admin-assisted password reset** (leverages physical proximity)

## Democratic Governance Implementation

Per ADR-005:

- Any member can create proposals with configurable voting rules
- Special proposal types (`ModeratorChange`, `AdminChange`, `MemberBan`) trigger automated system changes
- Vote outcomes are automatically enforced - admins cannot override
- Support both transparent (public votes) and private (anonymous) voting
- Communities configure their own governance defaults

## Content Model and Features

Per ADR-006 and ADR-007:

**Unified Content Model:**
- Single `Posts` table with `PostType` discriminator (Share, Thread, MarketplaceSell, MarketplaceBuy, Ask, Offer, Proposal)
- Type-specific metadata stored in JSON column (`TypeMetadata`)
- Comments support nested threading (up to 3 levels deep)
- Emoji reactions instead of simple "likes" for positive engagement
- @mentions tracked for notifications

**Content Formatting:**
- GitHub-Flavored Markdown for all text content
- Headlines available in Threads and Proposals (not Shares)
- Server-side rendering with sanitization to prevent XSS

**Media Handling:**
- Browser-side image processing (resize, convert to JPEG at ~75% quality)
- Configurable media limits per instance (default: 10 images per post)
- Support all file types for attachments
- Storage backend configurable: filesystem (default) or S3-compatible cloud storage
- Media URLs stored in JSON arrays (`Posts.MediaUrls`, `Comments.MediaUrls`)

**Content Editing:**
- No time limits for editing posts/comments
- Special handling for Proposals: cannot edit while "Open for Vote" or after "Resolved"
- Edit tracking via `IsEdited` flag and `UpdatedAt` timestamp
- Full edit history optional (future enhancement)

**Link Previews:**
- In-memory cache (24-hour duration)
- Fetch Open Graph metadata on-demand
- No database persistence (rebuilds after server restart)

## Deployment and Configuration

Per ADR-008:

**Deployment Options:**
- **Docker Compose (recommended):** Multi-container architecture with separate Web, API, and Litestream containers
- **Standalone binaries:** Platform-specific .NET executables for non-Docker deployments

**Configuration Layers:**
1. **Environment variables:** Infrastructure settings (database path, HTTPS, storage backend, media limits)
2. **Setup wizard:** Community settings (name, admin account, logos) - stored in database
3. **Instance config file:** Deployment-specific overrides (HTTP fallback, maintenance mode)

**Initial Setup:**
- Web-based setup wizard on first startup
- Guides operators through community identity, admin creation, storage config, and replication setup
- Minimal pre-configuration required (works with sensible defaults)

**HTTPS/TLS:**
- Built-in HTTPS with automatic Let's Encrypt certificates
- Custom certificate support for advanced deployments
- HTTP fallback configurable for isolated networks

**Upgrades:**
- Pull new Docker image and restart containers (Docker deployments)
- Replace binary and restart (standalone deployments)
- Database migrations run automatically on startup
- Manual rollback process using previous image/binary and Litestream backup

**Component Modularity:**
- Operators can run Web-only, API-only, or both
- Litestream runs when at least one component is active
- Blazor Web connects directly to SQLite (does not use API)
- API critical for future mobile apps

## Implementation Phases

When beginning implementation, follow this sequence:

**Phase 1: Core + Web MVP**
1. Set up solution structure (Core, Web projects)
2. Configure SQLite with EF Core + ASP.NET Core Identity
3. Implement basic authentication (login, registration)
4. Build core domain models (Members, Posts, Comments)
5. Create Blazor Server UI for social features
6. Integrate Litestream for local filesystem replication

**Phase 2: Democratic Governance**
1. Implement proposal and voting system
2. Add special proposal types with automated enforcement
3. Build governance configuration UI
4. Add invite code system

**Phase 3: API Layer**
1. Build REST API endpoints
2. Implement JWT authentication
3. Add Swagger/OpenAPI documentation
4. Enable modular deployment options

**Phase 4: Advanced Features**
1. Add cloud storage replication targets (Azure Blob, S3)
2. Implement peer node SFTP replication
3. Build monitoring for replication lag
4. Add mobile apps (.NET MAUI)

## Development Commands

When the .NET solution exists, common commands will be:

```bash
# Build the solution
dotnet build

# Run Blazor Server web app
dotnet run --project Groundwork.Web

# Run API only
dotnet run --project Groundwork.Api

# Run EF Core migrations
dotnet ef migrations add MigrationName --project Groundwork.Core
dotnet ef database update --project Groundwork.Web

# Run tests
dotnet test

# Publish for deployment
dotnet publish -c Release --self-contained -r linux-x64
```

## Testing Strategy

- Focus on business logic in `Groundwork.Core` (high unit test coverage)
- Integration tests for database operations (EF Core + SQLite in-memory)
- End-to-end tests for democratic governance (proposal outcomes)
- Test failover scenarios (restore from Litestream replica)

## Documentation Standards

- Document all ADRs using the provided [template](docs/adrs/template.md)
- **ADRs document decisions, not implementations** - do NOT include code examples or implementation details in ADRs
- Use XML doc comments for public APIs
- Maintain README.md for project overview, keep CLAUDE.md for implementation guidance

## Important Constraints

- **No external auth providers** in MVP (community ownership principle)
- **No automatic failover** (manual process acceptable for 5-50 users)
- **SQLite single-writer** is acceptable (single-primary architecture)
- **Must work offline** (isolated network scenarios, post-disaster resilience)
- **GPL v3 license** - all modifications must remain open source

## Future Considerations

These are planned but not yet implemented:

- Mobile apps consuming the REST API (.NET MAUI)
- Multi-target Litestream replication (cloud + peer nodes)
- Offline-first patterns for Blazor (service workers, local storage)
- Advanced voting methods (ranked choice, quadratic voting)
- Two-factor authentication via ASP.NET Core Identity
- Replication monitoring and alerting

## License

GNU General Public License v3.0 - ensures community infrastructure remains free and open. Any modifications must also be GPL v3.