# 001. Data Replication Strategy

## Status

Accepted

## Context

Groundwork Commons is designed for hyperlocal communities (5-50 members) who need resilient social infrastructure. The primary technical risk is data loss when a community member who operates the primary node becomes unavailable—whether due to hardware failure, leaving the community, losing moderation status, or infrastructure disruption (including post-disaster scenarios where cloud infrastructure may be unavailable).

### Problem Statement
We need a data replication strategy that:
- Prevents data loss when the primary node operator leaves or becomes unavailable
- Enables community continuity by allowing another member to spin up a new primary instance
- Works in both cloud-connected and isolated network scenarios
- Remains simple enough for small communities to operate without dedicated technical expertise
- Scales appropriately for 5-50 users (not millions)

### Constraints
- Small community scale (5-50 users)
- Node operators have varying technical expertise
- Must support both cloud infrastructure and isolated/mesh networks
- Post-disaster resilience is a design consideration
- Manual failover is acceptable; automatic failover is not required
- Operational simplicity is prioritized over complex distributed consensus

### Alternatives Considered

**Multi-Master Replication (CouchDB, Riak, etc.)**
- Would allow writes to any node with automatic conflict resolution
- Adds significant complexity for conflict handling
- Overkill for 5-50 users with infrequent failover scenarios
- Rejected: Unnecessary complexity for the problem we're solving

**Active-Standby with Automatic Failover (Raft, etcd, etc.)**
- Provides automatic leader election and failover
- Requires persistent network connectivity between nodes
- Adds operational complexity (cluster management, split-brain scenarios)
- Rejected: Fails in isolated network scenarios; too complex for small communities

**Manual Backup/Restore (Periodic Database Dumps)**
- Simple to understand and implement
- Risk of data loss between backup intervals
- Manual restore process could be slow during crisis
- Rejected: Point-in-time recovery gap is unacceptable

**Single-Primary with Continuous Replication (Chosen Approach)**
- One authoritative node handles all writes
- Continuous streaming replication to backup locations
- Manual failover when needed (spin up new primary from replica)
- Balances simplicity with data protection

## Decision

We will implement a **single-primary architecture with continuous replication** using:

1. **Primary Node Architecture:**
   - One designated primary node serves as the authoritative source
   - All write operations occur on the primary
   - Primary URL is shared among community members

2. **Replication Mechanism:**
   - Continuous streaming replication from primary to backup locations
   - Replication frequency is configurable (real-time to periodic) to support lower-end hardware
   - Replica lag monitoring to ensure backups are current

3. **Backup Storage Locations (Configurable):**
   - **Cloud storage** (Azure Blob Storage, S3, etc.) for normal operations
   - **Other community members' nodes** for redundancy and isolated network scenarios
   - **Local filesystem/SFTP** for post-disaster or mesh network deployments
   - Communities can configure multiple backup targets simultaneously

4. **Failover Process (Manual):**
   - When primary becomes unavailable, a community member:
     1. Retrieves latest database replica from backup location
     2. Spins up new node instance with replicated data
     3. Announces new primary URL through alternate communication channel
   - No automatic leader election or consensus protocol required

5. **Data Consistency Model:**
   - **Primary:** Strongly consistent (all writes to single source)
   - **Replicas:** Eventually consistent (streaming replication with configurable lag)
   - Acceptable replication lag: Seconds to minutes depending on hardware constraints

## Consequences

### Positive Consequences

- **Simple operational model:** Community members understand "one primary, multiple backups"
- **Data loss prevention:** Continuous replication ensures minimal data loss (only changes since last replication)
- **Disaster resilience:** Works in isolated networks and post-disaster scenarios
- **No split-brain complexity:** Single primary eliminates conflicting writes
- **Configurable replication targets:** Supports both cloud and local network scenarios
- **Resource efficient:** Single-primary requires less hardware than multi-master
- **Scalable replication frequency:** Can tune for hardware capabilities

### Negative Consequences

- **Manual failover required:** Community must coordinate to spin up new primary (acceptable for 5-50 users)
- **Primary is single point of write availability:** If primary goes down, no new writes until failover (read-only access to replicas possible)
- **Replication lag risk:** Very recent changes may be lost if primary fails between replication intervals
- **URL change on failover:** Members must learn new URL when primary changes (mitigated by alternate communication)
- **No automatic load balancing:** All traffic goes to single primary (acceptable for small scale)

### Neutral Consequences

- **Community must designate backup operators:** Need trusted members willing to run replica nodes or maintain backup storage
- **Monitoring requirements:** Should monitor replication lag and backup health
- **Documentation needs:** Must document failover procedures for non-technical operators
- **Backup retention policy:** Need to decide how long to keep historical replicas
- **Authentication across failover:** Member credentials must be in replicated data to survive failover

---

## Notes

### Related ADRs
- ADR-002: Database Technology Selection (implements this replication strategy)
- Future ADR: Node Operator Election and Designation Process
- Future ADR: Monitoring and Alerting Strategy

### Implementation Considerations
- Replication frequency should default to real-time but be configurable via environment variable
- Backup locations should be configurable via config file (support multiple simultaneous targets)
- Should implement health check endpoint for monitoring replication lag
- Consider implementing "replica read-only mode" for temporary access during primary downtime

### References
- [Litestream](https://litestream.io/) - Continuous SQLite replication tool
- CAP Theorem considerations: We choose Consistency and Partition Tolerance over Availability during failover
- Recovery Point Objective (RPO): Acceptable data loss = replication interval (seconds to minutes)
- Recovery Time Objective (RTO): Manual failover time = minutes to hours (acceptable for community scale)

### Timeline
- **Phase 1:** Implement single-primary with local filesystem replication
- **Phase 2:** Add cloud storage replication support
- **Phase 3:** Add multi-target replication (cloud + peer nodes)
- **Phase 4:** Build monitoring and alerting for replication health