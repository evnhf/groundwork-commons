# 002. Database Technology Selection

## Status

Accepted

## Context

Following ADR-001's decision to use single-primary continuous replication, we need to select a database technology that:
- Supports the .NET ecosystem
- Works well with continuous replication to backup locations
- Handles 5-50 users efficiently
- Provides both relational and document-oriented querying
- Remains simple to deploy and operate for community members
- Supports single-file or embedded deployment models

### Problem Statement
We need a database that can:
- Store social platform data (posts, comments, members, permissions, moderation actions)
- Support JSON-based document storage where appropriate (flexible schema for user-generated content)
- Use relational constraints where needed (referential integrity for permissions, member relationships)
- Replicate continuously to backup locations (cloud storage, peer nodes, local filesystems)
- Run on modest hardware (community member laptops, low-end VPS)
- Be restored quickly from backup for failover scenarios

### Constraints
- Must work well in .NET ecosystem (C#, ASP.NET Core)
- Single-file or embedded deployment preferred (simplifies backup/restore)
- No separate database server to manage (reduces operational complexity)
- Must support continuous replication to file-based targets
- Needs to handle both structured relational data and semi-structured JSON documents
- Should be license-compatible with GPL v3

### Alternatives Considered

**PostgreSQL with Logical Replication**
- Pros: Mature, excellent JSONB support, strong .NET support (Npgsql, EF Core)
- Pros: Built-in logical replication, battle-tested
- Cons: Requires separate database server process
- Cons: More complex to deploy and restore for non-technical users
- Cons: Overkill for 5-50 users
- Rejected: Operational complexity outweighs benefits at this scale

**LiteDB (Document Database)**
- Pros: .NET native, MongoDB-like API, single-file
- Pros: Document-first design, simple deployment
- Cons: Would require custom-built replication mechanism
- Cons: Less mature than SQLite, smaller ecosystem
- Cons: No built-in relational constraints
- Rejected: Building custom replication adds complexity; SQLite ecosystem is more mature

**Marten (Event Sourcing + Document DB on PostgreSQL)**
- Pros: Excellent .NET integration, event sourcing capabilities
- Pros: Built on PostgreSQL (mature replication)
- Cons: Requires PostgreSQL server (operational complexity)
- Cons: Event sourcing adds conceptual complexity we don't need
- Cons: Overkill for simple social platform data model
- Rejected: Unnecessary complexity for our use case

**CouchDB**
- Pros: Built-in multi-master replication protocol
- Pros: HTTP-based, mature conflict resolution
- Cons: Separate server process to run
- Cons: .NET is not primary ecosystem (requires HTTP client libraries)
- Cons: Multi-master capabilities unnecessary (we chose single-primary in ADR-001)
- Rejected: Operational complexity and language mismatch

**SQLite with Litestream (Chosen Approach)**
- Pros: Battle-tested, single-file, embeddable, no server process
- Pros: Excellent .NET support (Microsoft.Data.Sqlite, EF Core, Dapper)
- Pros: JSON1 extension provides document querying capabilities
- Pros: Litestream provides continuous replication to multiple targets (S3, Azure Blob, SFTP, filesystem)
- Pros: Simple backup/restore (copy .db file)
- Pros: Public domain license (compatible with GPL v3)
- Cons: Single-writer limitation (acceptable for single-primary architecture)
- Chosen: Best fit for scale, simplicity, and replication requirements

## Decision

We will use **SQLite with the JSON1 extension** as our database, with **Litestream** for continuous replication.

### Database: SQLite with JSON1 Extension

1. **Core Database:**
   - SQLite 3 (embedded, single-file database)
   - JSON1 extension enabled for JSON querying capabilities
   - Write-Ahead Logging (WAL) mode for better concurrency

2. **Data Modeling Approach:**
   - **Relational tables** for structured data requiring referential integrity:
     - Members, node operators, permissions, roles
     - Audit logs, moderation actions
   - **JSON columns** for semi-structured, flexible data:
     - Post content and metadata
     - User profiles and custom fields
     - Settings and configuration
   - Hybrid approach: Use relational when structure is known and stable, JSON when flexibility is needed

3. **.NET Integration:**
   - **Microsoft.Data.Sqlite** for low-level access
   - **EF Core SQLite provider** for ORM capabilities
   - **Dapper** for performance-critical queries if needed
   - Use EF Core migrations for schema versioning

### Replication: Litestream

1. **Replication Tool:**
   - [Litestream](https://litestream.io/) - Continuous SQLite replication tool
   - Runs as sidecar process alongside .NET application
   - Streams WAL (Write-Ahead Log) changes to backup locations

2. **Replication Targets (Configurable):**
   - **Cloud storage:** Azure Blob Storage, Amazon S3, Google Cloud Storage
   - **SFTP:** Other community members' nodes or servers
   - **Local filesystem:** Mounted network drives, USB storage
   - Support multiple simultaneous targets for redundancy

3. **Replication Frequency:**
   - Real-time streaming of WAL segments (sub-second to seconds)
   - Configurable via Litestream settings to support lower-end hardware
   - Retention policy: Maintain last N days of WAL history for point-in-time recovery

4. **Restore Process:**
   - `litestream restore` command pulls latest replica from backup location
   - Single command to get community back online with new primary
   - Can restore to specific point in time if needed

### Deployment Model

```
┌─────────────────────────────────┐
│     Primary Node Server         │
│                                 │
│  ┌──────────────────┐           │
│  │  ASP.NET Core    │           │
│  │  Application     │           │
│  └────────┬─────────┘           │
│           │                     │
│           ▼                     │
│  ┌──────────────────┐           │
│  │  SQLite DB       │           │
│  │  (data.db)       │◄──────┐   │
│  └──────────────────┘       │   │
│                             │   │
│  ┌──────────────────┐       │   │
│  │  Litestream      ├───────┘   │
│  │  (replication)   │           │
│  └────────┬─────────┘           │
└───────────┼─────────────────────┘
            │
            ▼
   ┌────────────────────┐
   │  Backup Locations  │
   ├────────────────────┤
   │ • Azure Blob       │
   │ • S3               │
   │ • Peer Node SFTP   │
   │ • Local Filesystem │
   └────────────────────┘
```

## Consequences

### Positive Consequences

- **Operational simplicity:** Single file database, no server to configure or manage
- **Excellent .NET integration:** First-class support via Microsoft.Data.Sqlite and EF Core
- **Hybrid data model:** Relational structure when needed, JSON flexibility when beneficial
- **Battle-tested reliability:** SQLite is one of the most tested and deployed databases globally
- **Trivial backup/restore:** Database is a single file that can be copied/restored easily
- **Continuous replication:** Litestream handles streaming replication to multiple targets automatically
- **Point-in-time recovery:** Can restore to any moment within retention window
- **Resource efficient:** Minimal memory and CPU overhead, perfect for 5-50 users
- **No licensing costs:** SQLite is public domain, Litestream is Apache 2.0
- **Portable:** Database file works across platforms (Windows, Linux, macOS)

### Negative Consequences

- **Single writer:** SQLite has single-writer limitation (acceptable for single-primary architecture from ADR-001)
- **No built-in authentication:** Unlike PostgreSQL, no database-level user permissions (handled at application layer)
- **Limited concurrency:** High write concurrency could be bottleneck (unlikely with 5-50 users)
- **Litestream dependency:** Adds external tool dependency (but well-maintained and simple)
- **JSON query performance:** JSON1 extension is less optimized than PostgreSQL's JSONB (acceptable at this scale)
- **No stored procedures:** Business logic must live in application layer (actually a positive for maintainability)

### Neutral Consequences

- **File size management:** Need to monitor database growth and implement retention policies
- **WAL checkpoint tuning:** May need to tune WAL checkpointing for optimal replication
- **Backup storage costs:** Cloud storage costs for replicas (minimal at small scale)
- **Litestream configuration:** Need to document Litestream setup for node operators
- **Schema migrations:** Must use EF Core migrations or custom migration tooling
- **Monitoring:** Need to monitor replication lag via Litestream metrics

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (this implements the chosen replication approach)
- Future ADR: Application Framework Selection (.NET/ASP.NET Core)
- Future ADR: Schema Design and Migration Strategy

### Implementation Details

**SQLite Configuration:**
```sql
-- Enable WAL mode for better concurrency and Litestream compatibility
PRAGMA journal_mode = WAL;

-- Load JSON1 extension
-- (Built-in with most SQLite distributions, including Microsoft.Data.Sqlite)

-- Example hybrid schema
CREATE TABLE members (
    id INTEGER PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    profile_data TEXT, -- JSON column for flexible profile fields
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

CREATE TABLE posts (
    id INTEGER PRIMARY KEY,
    author_id INTEGER NOT NULL,
    content TEXT NOT NULL, -- JSON column for rich content (text, images, metadata)
    created_at INTEGER NOT NULL,
    FOREIGN KEY (author_id) REFERENCES members(id)
);

-- JSON querying example
SELECT * FROM members
WHERE json_extract(profile_data, '$.bio') LIKE '%gardening%';
```

**Litestream Configuration Example:**
```yaml
# /etc/litestream.yml
dbs:
  - path: /var/lib/groundwork/data.db
    replicas:
      - type: azure
        bucket: groundwork-backup
        path: neighborhood-name

      - type: sftp
        host: peer-node.local
        user: replication
        path: /backup/data.db

      - type: file
        path: /mnt/backup/data.db

    # Replication frequency (tune for hardware)
    sync-interval: 10s # Can increase for lower-end hardware
```

### .NET Package References
- `Microsoft.Data.Sqlite` - Low-level SQLite driver
- `Microsoft.EntityFrameworkCore.Sqlite` - EF Core provider
- `Dapper` (optional) - Micro-ORM for performance-critical queries

### References
- [SQLite Documentation](https://www.sqlite.org/docs.html)
- [SQLite JSON1 Extension](https://www.sqlite.org/json1.html)
- [Litestream Documentation](https://litestream.io/)
- [Microsoft.Data.Sqlite Documentation](https://learn.microsoft.com/en-us/dotnet/standard/data/sqlite/)
- [EF Core SQLite Provider](https://learn.microsoft.com/en-us/ef/core/providers/sqlite/)

### Performance Considerations
- SQLite handles tens of thousands of transactions per second on modern hardware
- With 5-50 users, concurrent write contention is extremely unlikely
- Read scaling is nearly unlimited (multiple readers can access simultaneously)
- Database file size: Estimate ~10-50MB for first year with active community (easily managed)

### Migration from SQLite (If Needed in Future)
If the project grows beyond SQLite's capabilities:
- EF Core abstracts database access, making migration to PostgreSQL relatively straightforward
- Export/import tooling can migrate data between databases
- JSON column data can map to PostgreSQL JSONB
- Decision point: 100+ active users or high write concurrency requirements

### Timeline
- **Phase 1:** Basic SQLite setup with relational schema
- **Phase 2:** Add JSON columns for flexible content storage
- **Phase 3:** Integrate Litestream with local filesystem replication
- **Phase 4:** Add cloud storage replication targets
- **Phase 5:** Implement peer node SFTP replication