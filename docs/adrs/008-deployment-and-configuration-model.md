# 008. Deployment and Configuration Model

## Status

Accepted

## Context

Groundwork Commons must be deployable by neighborhood operators with varying levels of technical expertise. The platform serves 5-50 members and should be simple to install, configure, and upgrade while maintaining the resilience and data sovereignty principles established in previous ADRs.

### Problem Statement

We need to define:
- How operators install and run the software (packaging, distribution)
- Docker container architecture and orchestration
- Litestream integration for database replication
- Configuration management (environment variables, files, database settings)
- HTTPS/TLS setup and certificate management
- Initial bootstrap and setup wizard flow
- Upgrade process and database migrations
- Backup and restore procedures

### Design Principles

1. **Operational Simplicity:** Easy to deploy for non-experts, powerful for advanced users
2. **Multiple Deployment Options:** Docker (most common) and standalone binaries (flexibility)
3. **Sensible Defaults:** Works out-of-the-box with minimal configuration
4. **Data Sovereignty:** All configuration and data stored locally or in operator-controlled locations
5. **Resilience:** Configuration supports the replication strategy from ADR-001
6. **Modularity:** Operators choose which components to run (API, Web, or both)
7. **Security by Default:** HTTPS encouraged but configurable for special cases

### Unique Requirements

- **Small-scale focus:** Optimized for single-server deployment, not Kubernetes clusters
- **Litestream integration:** Replication must be seamless and automatic
- **Setup wizard:** Non-technical operators need guided initial setup
- **Isolated network support:** Must work without internet access
- **Manual failover:** Restoration process should be documented and simple
- **Modular components:** Web and API run as separate containers/processes

### Alternatives Considered

**Source Code Compilation Only**
- Operators clone repo, build from source
- Pros: Maximum flexibility, always latest code
- Cons: Requires .NET SDK, build tools, technical expertise
- Rejected: Too high barrier for neighborhood operators

**Single Monolithic Container**
- All components (Web, API, DB, Litestream) in one container
- Pros: Simplest deployment (one container to run)
- Cons: Not modular, hard to scale components independently
- Rejected: Violates modularity principle from ADR-003

**Platform-as-a-Service (Hosted Solution)**
- Provide hosted version, operators just sign up
- Pros: Simplest for users, no infrastructure management
- Cons: Contradicts data sovereignty, requires ongoing operational costs
- Rejected: Incompatible with community ownership principles

**Kubernetes/Helm Charts**
- Provide Kubernetes deployment manifests
- Pros: Industry-standard orchestration, auto-scaling
- Cons: Massive complexity overkill for 5-50 users, requires K8s cluster
- Rejected: Too complex for target audience

**Chosen: Docker Compose + Standalone Binaries**
- **Docker Compose:** Multi-container architecture (Web, API, Litestream as separate containers)
- **Standalone Binaries:** Platform-specific executables for non-Docker deployments
- Pros: Simple Docker deployment, flexibility for advanced users, modular
- Chosen: Best balance of simplicity, modularity, and accessibility

## Decision

We will provide **two deployment options** with **Docker Compose as the primary method** and **standalone .NET binaries as an alternative**. Configuration will use **environment variables for infrastructure** and **a setup wizard for community settings**. HTTPS will be **built-in and enabled by default** but configurable for special deployments.

### 1. Deployment Options

**Option 1: Docker Compose (Recommended)**

Multi-container architecture:

```yaml
services:
  groundwork-web:
    image: groundwork/commons-web:latest
    container_name: groundwork-web
    depends_on:
      - groundwork-api
    volumes:
      - ./data:/app/data
      - ./media:/app/media
    environment:
      - API_URL=http://groundwork-api:5000
      - DATABASE_PATH=/app/data/groundwork.db
      - HTTPS_ENABLED=true
      - HTTPS_PORT=443
    ports:
      - "443:443"
      - "80:80"

  groundwork-api:
    image: groundwork/commons-api:latest
    container_name: groundwork-api
    volumes:
      - ./data:/app/data
      - ./media:/app/media
    environment:
      - DATABASE_PATH=/app/data/groundwork.db
    ports:
      - "5000:5000"

  litestream:
    image: litestream/litestream:latest
    container_name: groundwork-litestream
    volumes:
      - ./data:/app/data
      - ./media:/app/media
      - ./litestream.yml:/etc/litestream.yml
    command: replicate
    depends_on:
      - groundwork-web
      - groundwork-api
```

**Database Volume:**
- SQLite database stored in `./data/groundwork.db` (volume mount)
- Shared across Web, API, and Litestream containers
- Persists when containers restart

**Media Volume:**
- Media files stored in `./media/` (volume mount)
- Shared across Web and API containers
- Persists when containers restart

**Option 2: Standalone .NET Binaries**

Platform-specific executables:
- Windows: `groundwork-web.exe`, `groundwork-api.exe`
- Linux: `groundwork-web`, `groundwork-api`
- macOS: `groundwork-web`, `groundwork-api`

**Installation:**
1. Download binary for platform
2. Extract to installation directory
3. Configure via `appsettings.json` or environment variables
4. Run executable: `./groundwork-web` or `./groundwork-api`
5. Install Litestream separately and configure

**Use Case:** Advanced users, non-Docker environments, development

### 2. Component Architecture

**Separate Containers/Processes:**

**groundwork-web (Blazor Server):**
- Serves web interface on port 443 (HTTPS) / 80 (HTTP)
- Connects directly to SQLite database
- Does NOT call API for data access (per ADR-003 note)
- Optional: Can be excluded if operator wants API-only deployment

**groundwork-api (ASP.NET Core Web API):**
- Serves REST API on port 5000
- Connects directly to SQLite database
- Important for future mobile apps
- Optional: Can be excluded if operator wants Web-only deployment

**litestream (Replication Service):**
- Continuously replicates SQLite database
- Runs only if at least one of Web or API is running
- Configured via `litestream.yml` file
- Replicates to cloud storage, peer nodes, or filesystem

**Conditional Litestream:**
- If both Web and API running: Litestream runs
- If only Web running: Litestream runs
- If only API running: Litestream runs
- If neither running: Litestream doesn't run
- Prevents duplicate replication when both components active

### 3. Litestream Integration

**Litestream Configuration:**

Operators configure replication via `litestream.yml`:

```yaml
dbs:
  - path: /app/data/groundwork.db
    replicas:
      # Cloud storage replica (S3-compatible)
      - type: s3
        bucket: my-community-backup
        path: groundwork
        region: us-west-2
        access-key-id: ${AWS_ACCESS_KEY_ID}
        secret-access-key: ${AWS_SECRET_ACCESS_KEY}

      # Peer node replica (SFTP)
      - type: sftp
        host: peer-node.local
        user: backup
        path: /backups/groundwork
        key-path: /etc/ssh/backup_key

      # Local filesystem replica
      - type: file
        path: /mnt/backup/groundwork
```

**Replication Strategy:**
- Default: Real-time replication (seconds)
- Configurable: Periodic replication for lower-end hardware
- Multiple targets: Cloud + peer + local simultaneously

**Filesystem Replication:**

Per ADR-007 decision and ADR-001 considerations:

**If Litestream supports filesystem replication:**
- Configure Litestream to replicate entire `/app/data` and `/app/media` directories
- Media and database replicate together

**If Litestream does NOT support filesystem replication:**
- Litestream only replicates SQLite database
- Separate mechanism needed for media:
  - **Option A:** Scheduled rsync/rclone to same backup locations
  - **Option B:** Application-level media sync
  - **Option C:** Use cloud storage for media (handles own redundancy)

**Implementation Note:** Determine Litestream filesystem capabilities during implementation and document accordingly.

### 4. Configuration Management

**Configuration Layers:**

**1. Infrastructure Configuration (Environment Variables)**

Set via Docker Compose `.env` file or system environment:

```bash
# Database
DATABASE_PATH=/app/data/groundwork.db

# HTTPS/TLS
HTTPS_ENABLED=true
HTTPS_PORT=443
HTTP_PORT=80
CERTIFICATE_PATH=/app/certs/cert.pem  # If using custom cert
CERTIFICATE_KEY_PATH=/app/certs/key.pem

# Storage Backend (from ADR-007)
STORAGE_BACKEND=filesystem  # or s3
STORAGE_PATH=/app/media
S3_BUCKET=
S3_REGION=
S3_ACCESS_KEY=
S3_SECRET_KEY=
S3_ENDPOINT=

# Media Limits (from ADR-007)
MAX_IMAGE_SIZE_MB=5
MAX_FILE_SIZE_MB=10
MAX_IMAGES_PER_POST=10

# Content Limits
MAX_POST_LENGTH=10000
MAX_COMMENT_LENGTH=2000

# Link Preview Cache
LINK_PREVIEW_CACHE_DURATION_HOURS=24
```

**Why Environment Variables:**
- Simple to configure in Docker Compose
- Standard practice for infrastructure settings
- Easy to override per deployment environment
- No UI needed for initial setup

**2. Community Settings (Database + Setup Wizard)**

Set via web-based setup wizard on first run:

- Community name
- Community description
- Admin username and password
- Timezone
- Community logo/banner (uploaded media)
- Invite code generation settings
- Governance defaults (from ADR-005)

**Why Database:**
- Needs to replicate with Litestream (survives failover)
- Includes uploaded media (logos)
- Easier to update via admin UI than config files

**3. Local Instance Configuration (Config File)**

Optional `instance.config.json` for deployment-specific settings:

```json
{
  "allowHttp": false,
  "enforceHttps": true,
  "setupWizardEnabled": true,
  "maintenanceMode": false
}
```

**Use Cases:**
- Override security settings for isolated networks (`allowHttp: true`)
- Disable setup wizard after initial configuration
- Enable maintenance mode during upgrades

### 5. HTTPS/TLS Configuration

**Default: Built-in HTTPS with Let's Encrypt**

**Automatic Certificate Generation:**
1. Operator configures domain name in environment variables
2. Application requests Let's Encrypt certificate on first startup
3. Certificate automatically renewed every 90 days
4. HTTP automatically redirects to HTTPS

**Configuration:**
```bash
HTTPS_ENABLED=true
DOMAIN_NAME=groundwork.myneighborhood.org
LETSENCRYPT_EMAIL=admin@myneighborhood.org
```

**Alternative: Custom Certificates**

Operators can provide their own certificates:
```bash
HTTPS_ENABLED=true
CERTIFICATE_PATH=/app/certs/cert.pem
CERTIFICATE_KEY_PATH=/app/certs/key.pem
```

**HTTP-Only Fallback (Configurable):**

For isolated networks or special deployments:

```json
// instance.config.json
{
  "allowHttp": true,
  "enforceHttps": false
}
```

**Use Cases:**
- Isolated networks with no domain name
- Testing/development environments
- Post-disaster mesh networks
- Behind reverse proxy handling TLS

**Security Trade-off:**
- HTTP allowed only when explicitly configured
- Operators understand security implications
- Acceptable for physically isolated communities

### 6. Initial Bootstrap and Setup Wizard

**First-Time Startup Flow:**

**1. Operator Starts Containers:**
```bash
docker-compose up -d
```

**2. Pre-Configuration (Optional):**
- Operator can pre-configure infrastructure settings in `.env`
- Minimal required: None (works with defaults)

**3. Application Detects Fresh Install:**
- Checks if database exists at `DATABASE_PATH`
- If not exists: Create empty database
- If exists but no admin user: Show setup wizard
- If exists with admin: Normal operation

**4. Setup Wizard UI (First Access):**

User navigates to `https://localhost` (or configured domain):

**Step 1: Welcome**
- Explain Groundwork Commons
- Confirm operator wants to proceed with setup

**Step 2: Community Identity**
- Community name (required)
- Community description
- Upload logo (optional)
- Timezone selection

**Step 3: Admin Account Creation**
- Username (required)
- Password (required, strength validation)
- Email (optional, for password reset)

**Step 4: Storage Configuration**
- Review configured storage backend (from env vars)
- Configure media upload limits
- Display storage path/bucket

**Step 5: Replication Setup**
- Review Litestream configuration (from `litestream.yml`)
- Show replication targets
- Test connectivity to backup locations

**Step 6: Governance Defaults**
- Set default proposal voting rules (from ADR-005)
- Configure invite code settings

**Step 7: Confirmation**
- Review all settings
- Confirm and initialize community

**5. Initialization:**
- Create admin user in database
- Save community settings to database
- Generate initial invite codes
- Start Litestream replication
- Redirect to community homepage

**Setup Wizard Notes:**
- Can be disabled after initial setup (`setupWizardEnabled: false`)
- Can be re-run if database is reset (not recommended in production)
- All steps except admin credentials have defaults

### 7. Upgrade Process

**Docker Deployments:**

**Upgrade Steps:**
1. Pull new image: `docker-compose pull`
2. Stop containers: `docker-compose down`
3. Start containers: `docker-compose up -d`
4. Database migrations run automatically on startup

**Database Migrations:**
- EF Core migrations execute on application startup
- Check current database version vs. code version
- Apply pending migrations automatically
- Log migration results

**Rollback Process:**
1. Stop containers: `docker-compose down`
2. Restore previous image version
3. Restore database from Litestream replica (if migration broke)
4. Start containers: `docker-compose up -d`

**Binary Deployments:**

**Upgrade Steps:**
1. Download new binary version
2. Stop running process
3. Replace old binary with new binary
4. Start new process
5. Database migrations run automatically on startup

**Rollback Process:**
1. Stop process
2. Restore previous binary version
3. Restore database from backup (if migration broke)
4. Start previous process

**Migration Safety:**
- Litestream creates replica before migrations
- Can restore to pre-migration state if upgrade fails
- Logs indicate migration success/failure

**No Automatic Rollback:**
- Operator manually reverts to previous version if needed
- Acceptable for small scale (5-50 users)
- Downtime measured in minutes

### 8. Backup and Restore

**Backup (Automatic via Litestream):**

**Continuous Replication:**
- Litestream automatically replicates database to configured targets
- No operator intervention required
- Replication lag typically seconds to minutes

**Manual Backup:**
```bash
# Docker
docker exec groundwork-litestream litestream restore -o /tmp/backup.db /app/data/groundwork.db

# Binary
litestream restore -o /tmp/backup.db /path/to/groundwork.db
```

**Restore (Manual Process):**

**Scenario: Primary Node Failure**

**Step 1: Retrieve Latest Replica**
```bash
# From cloud storage (S3)
litestream restore -replica s3 /path/to/groundwork.db

# From peer node (SFTP)
litestream restore -replica sftp /path/to/groundwork.db

# From local filesystem
cp /mnt/backup/groundwork/db /path/to/groundwork.db
```

**Step 2: Copy Media Files**
- If media stored on filesystem: Restore from backup location
- If media stored in cloud: Already available, no action needed

**Step 3: Start New Primary Node**
```bash
# Docker
docker-compose up -d

# Binary
./groundwork-web
```

**Step 4: Announce New Primary URL**
- Communicate new URL to community via alternate channel (phone, text, email)
- Members update bookmarks

**Step 5: Resume Operations**
- Community resumes normal use
- Litestream begins replicating from new primary

**Restore Notes:**
- Data loss limited to changes since last replication (seconds to minutes)
- Restoration time: Minutes to hours (acceptable for community scale)
- Manual coordination required (no automatic failover)

### 9. Deployment Modularity

**Scenario 1: Full Stack (Web + API)**

```yaml
# docker-compose.yml
services:
  groundwork-web:
    # ... (enabled)
  groundwork-api:
    # ... (enabled)
  litestream:
    # ... (enabled, replicates for both)
```

**Use Case:** Support both web users and future mobile apps

**Scenario 2: Web Only**

```yaml
# docker-compose.yml
services:
  groundwork-web:
    # ... (enabled)
  # groundwork-api: (commented out)
  litestream:
    # ... (enabled, replicates database)
```

**Use Case:** Community only uses web interface, no mobile apps

**Scenario 3: API Only**

```yaml
# docker-compose.yml
services:
  # groundwork-web: (commented out)
  groundwork-api:
    # ... (enabled)
  litestream:
    # ... (enabled, replicates database)
```

**Use Case:** Mobile-only community, minimal resource usage

### 10. Health Checks and Monitoring

**Health Check Endpoints:**

**Web:**
- `GET /health` - Returns 200 if application healthy
- `GET /health/db` - Returns 200 if database accessible

**API:**
- `GET /api/health` - Returns 200 if application healthy
- `GET /api/health/db` - Returns 200 if database accessible

**Litestream:**
- `GET /metrics` - Returns replication lag, last backup time
- `GET /healthz` - Returns 200 if replication active

**Docker Compose Integration:**
```yaml
groundwork-web:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost/health"]
    interval: 30s
    timeout: 10s
    retries: 3
```

**Monitoring (Optional):**
- Operators can integrate health endpoints with monitoring tools
- Simple uptime checks sufficient for small scale
- Advanced: Prometheus metrics for replication lag

## Consequences

### Positive Consequences

- **Simple deployment:** Docker Compose requires minimal setup, works out-of-box
- **Flexibility:** Binary option for non-Docker users and advanced configurations
- **Modular architecture:** Separate containers for Web, API, Litestream
- **Automatic migrations:** Database updates seamless on upgrade
- **Setup wizard:** Non-technical operators guided through initial configuration
- **Built-in HTTPS:** Let's Encrypt integration makes TLS simple
- **Configurable security:** HTTP fallback for isolated networks
- **Backup simplicity:** Litestream automatic, restore process documented
- **Environment variables:** Standard infrastructure configuration approach
- **Resilient:** Configuration survives failover (stored in replicated database)

### Negative Consequences

- **Docker knowledge required:** Most operators need basic Docker/Compose familiarity
- **Manual upgrades:** No automatic update mechanism (operator must pull/restart)
- **Manual failover:** Restoration requires operator intervention
- **URL changes on failover:** Community must communicate new URL
- **Litestream dependency:** Replication relies on external tool (mitigated by maturity)
- **Migration risk:** Database migrations could fail (rollback available)
- **No zero-downtime upgrades:** Restart required for new versions (acceptable for small scale)

### Neutral Consequences

- **Setup wizard required:** Initial configuration takes ~10 minutes
- **Config file complexity:** Multiple configuration layers (env, database, file)
- **Health check integration:** Operators must set up monitoring if desired
- **Backup testing:** Operators should periodically test restore process
- **Documentation maintenance:** Need clear docs for deployment, upgrade, restore
- **Platform-specific binaries:** Must build/distribute for Windows, Linux, macOS

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (Litestream implementation)
- ADR-002: Database Technology Selection (SQLite deployment)
- ADR-003: Application Framework and Technology Stack (modular components)
- ADR-004: Authentication and Authorization Model (admin account creation)
- ADR-007: Content Features and Capabilities (storage backend configuration)

### Implementation Phases

**Phase 1: Docker Compose Deployment (MVP)**
- Build Docker images for Web and API
- Create reference `docker-compose.yml`
- Environment variable configuration
- Basic setup wizard (admin account, community name)
- Manual Litestream integration

**Phase 2: Standalone Binary Distribution**
- Publish platform-specific executables
- Document binary deployment process
- Platform-specific installation guides

**Phase 3: HTTPS/TLS Integration**
- Implement Let's Encrypt automatic certificates
- Custom certificate support
- HTTP fallback configuration

**Phase 4: Enhanced Setup Wizard**
- Full wizard flow (all steps)
- Litestream configuration validation
- Storage backend testing
- Governance defaults

**Phase 5: Upgrade Automation**
- Automatic migration execution
- Migration rollback support
- Upgrade notification in admin UI

**Phase 6: Monitoring and Health Checks**
- Health check endpoints
- Prometheus metrics export
- Replication lag alerting
- Admin dashboard for system health

### Common Operations

**Installing with Docker Compose:**
```bash
# 1. Download docker-compose.yml
wget https://groundwork.org/docker-compose.yml

# 2. Configure environment (optional)
cp .env.example .env
nano .env

# 3. Configure Litestream (optional)
cp litestream.yml.example litestream.yml
nano litestream.yml

# 4. Start containers
docker-compose up -d

# 5. Access setup wizard
open https://localhost
```

**Upgrading:**
```bash
# 1. Pull latest images
docker-compose pull

# 2. Restart containers
docker-compose down && docker-compose up -d

# 3. Check logs for migration success
docker-compose logs -f
```

**Restoring from Backup:**
```bash
# 1. Stop existing instance
docker-compose down

# 2. Restore database
docker run --rm -v $(pwd)/data:/app/data litestream/litestream restore -replica s3 /app/data/groundwork.db

# 3. Restore media (if filesystem storage)
rsync -av backup-server:/backups/media/ ./media/

# 4. Start new instance
docker-compose up -d
```

**Checking Replication Status:**
```bash
docker exec groundwork-litestream litestream snapshots /app/data/groundwork.db
```

### Configuration Examples

**Minimal `.env` (Uses Defaults):**
```bash
# Nothing required! Defaults work.
```

**Production `.env` (Full Configuration):**
```bash
# Domain and HTTPS
DOMAIN_NAME=groundwork.myneighborhood.org
LETSENCRYPT_EMAIL=admin@myneighborhood.org
HTTPS_ENABLED=true

# Storage
STORAGE_BACKEND=s3
S3_BUCKET=myneighborhood-media
S3_REGION=us-west-2
S3_ACCESS_KEY=AKIA...
S3_SECRET_KEY=wJalrXUtn...

# Media Limits
MAX_IMAGE_SIZE_MB=10
MAX_FILE_SIZE_MB=20
MAX_IMAGES_PER_POST=15

# Content Limits
MAX_POST_LENGTH=20000
MAX_COMMENT_LENGTH=5000
```

**Development `.env` (HTTP, Local Storage):**
```bash
# Allow HTTP for local development
HTTPS_ENABLED=false
HTTP_PORT=8080

# Local filesystem storage
STORAGE_BACKEND=filesystem
STORAGE_PATH=/app/media
```

### References
- [Docker Compose Documentation](https://docs.docker.com/compose/) - Container orchestration
- [Litestream Documentation](https://litestream.io/guides/) - Database replication
- [Let's Encrypt](https://letsencrypt.org/) - Free TLS certificates
- [.NET Self-Contained Deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/) - Binary distribution
- [EF Core Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/) - Database migration management

### Future Enhancements
- **Automatic updates:** Check for new versions, prompt operator to upgrade
- **One-click restore:** Admin UI for restoration from backup
- **Backup verification:** Automated restore testing to verify backup integrity
- **Multi-region deployment:** Deploy API in multiple regions for lower latency
- **Container orchestration:** Kubernetes support for advanced deployments
- **Configuration templates:** Pre-built configs for common scenarios
- **Installation scripts:** One-command installation for common platforms
- **Admin CLI:** Command-line tools for operators (user management, backup, etc.)
- **Telemetry (opt-in):** Anonymous usage stats to improve platform
- **Update notifications:** Check for security updates and notify operators