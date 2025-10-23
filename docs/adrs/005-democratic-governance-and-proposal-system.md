# 005. Democratic Governance and Proposal System

## Status

Accepted

## Context

Groundwork Commons is built on the principle that technical infrastructure should enable democratic community governance. Unlike traditional social platforms where moderation and policy decisions are made by corporate employees or platform owners, Groundwork empowers neighborhood communities to make collective decisions through formal proposal and voting mechanisms.

The proposal system must balance:
- **Democratic legitimacy:** Members can propose and vote on important decisions
- **Flexibility:** Different communities want different governance rules
- **Automation:** Vote outcomes can trigger system changes (e.g., role assignments)
- **Transparency:** Voting records preserve community accountability
- **Simplicity:** Non-technical members can participate easily

### Problem Statement
We need to define:
- How proposals are created and configured
- What types of decisions can be automated via proposals
- How voting is conducted (mechanisms, transparency, time limits)
- How vote outcomes are calculated and enforced
- Whether each community can define its own governance rules
- How to prevent vote manipulation while preserving accessibility

### Constraints
- Must integrate with existing role system (ADR-004)
- Stored in SQLite database, replicated across nodes (ADR-001, ADR-002)
- Accessible via Blazor web interface (ADR-003)
- 5-50 members per community (small-scale voting)
- Members may have varying levels of technical sophistication
- No external services (must work in isolated networks)

### Democratic Principles
- **Member agency:** Any member can propose changes
- **Collective decision-making:** Important decisions require community consensus
- **Accountability:** Voting records provide transparency (configurable per proposal)
- **Flexibility:** Communities self-govern their own rules
- **Automation:** Technical systems enforce democratic outcomes (no admin override)

### Alternatives Considered

**No Formal Voting System (Admin-Only Decisions)**
- Pros: Simple to implement
- Cons: Contradicts democratic governance principles
- Cons: Creates power imbalance (admins control everything)
- Rejected: Fails core mission of community ownership

**Off-Platform Voting (External Poll Services)**
- Pros: No need to build voting system
- Cons: Requires external internet connectivity
- Cons: Vote outcomes not enforceable in system
- Cons: Corporate dependency (Doodle, SurveyMonkey, etc.)
- Rejected: Doesn't integrate with platform governance

**Blockchain-Based Voting (Smart Contracts)**
- Pros: Immutable vote records, cryptographic verification
- Cons: Massive complexity overhead
- Cons: Requires blockchain infrastructure
- Cons: Overkill for 5-50 person neighborhoods
- Rejected: Unnecessary complexity for scale

**Flexible Proposal System with Configurable Rules (Chosen Approach)**
- Pros: Proposers configure voting rules per proposal
- Pros: Supports both transparent and anonymous voting
- Pros: Enables special proposal types with automated outcomes
- Pros: Communities can establish governance conventions over time
- Pros: Simple database storage, no external dependencies
- Chosen: Best balance of flexibility, simplicity, and democratic principles

## Decision

We will implement a **flexible proposal and voting system** where:
1. Any member can create a proposal
2. Proposal creator configures voting rules (type, transparency, duration, thresholds)
3. Members cast votes according to proposal rules
4. System calculates outcome when voting period ends
5. Special proposal types trigger automated system changes (e.g., role assignments)
6. Communities develop governance conventions through practice

### 1. Proposal Model

### 2. Vote Model

### 3. Proposal Creation Flow

**Step 1: Member Initiates Proposal**
- Navigate to "Create Proposal" page
- Can convert existing discussion thread into proposal (optional feature)

**Step 2: Configure Proposal**
Form fields:
- **Title:** Short summary (e.g., "Remove Steve from Moderator Role")
- **Description:** Detailed explanation with context (Markdown supported)
- **Proposal Type:** Select from dropdown (General, ModeratorChange, etc.)
- **Voting Method:** Yes/No, Yes/No/Abstain, Multiple Choice
- **Voting Transparency:** Public (show votes) or Private (show counts only)
- **Voting Duration:** Start time, end time (e.g., 7 days from now)
- **Passing Threshold:** Percentage required to pass (default: 50%, community can customize)
- **Minimum Participation:** Minimum number of voters (default: none)

**Step 3: Special Proposal Configuration**
If `ProposalType` is special (e.g., `ModeratorChange`):
- **Target User:** Select member affected by proposal
- **Action:** Add role or Remove role
- **Role:** Which role (Moderator, Admin)

Metadata stored as JSON:
```json
{
  "targetUserId": "user-123",
  "action": "remove",
  "role": "Moderator"
}
```

**Step 4: Submit**
- Proposal created with status `Draft`
- Creator can preview, edit, or activate
- Once activated, status changes to `Active` and voting begins

### 4. Voting Process

**Casting a Vote:**
1. Member navigates to active proposal
2. Reads description and configuration
3. Selects vote choice (Yes, No, Abstain, or option)
4. Confirms vote (cannot change once cast)
5. Vote stored in database

**Vote Validation:**
- One vote per member per proposal (enforced by database constraint)
- Cannot vote on expired proposals
- Cannot vote on own proposals (optional rule, configurable)

**Real-Time Updates:**
- SignalR (Blazor Server) pushes vote count updates to connected clients
- Live progress bar shows participation rate
- If transparent voting: Live list of who voted what

**Vote Tally:**

### 5. Outcome Calculation

**When Voting Ends (Automated):**
Background job checks for proposals where `VotingEndsAt <= DateTime.UtcNow` and `Status = Active`.

**Calculate Results:**

**Enforce Outcome:**

### 6. Governance Configuration

**Community-Specific Settings:**
Each Groundwork Commons instance can configure defaults via settings table:

**Admin Configuration UI:**
- Admins can modify these settings via admin panel
- Changes apply to future proposals (not retroactive)
- Communities establish norms over time (e.g., "We always use 2/3 majority for role changes")

### 7. Member Onboarding and Node Operator Election

**Initial Node Operator:**
- **First user to set up instance becomes initial Admin**
- Automatically assigned Admin role upon system initialization
- No election needed for bootstrap

**Subsequent Node Operators:**
- Communities decide their own process
- Typically: Admin role = node operator
- To add new node operator:
  1. Create `AdminChange` proposal to grant Admin role to new member
  2. Community votes
  3. If passed, new member becomes Admin (and can run replica node)

**Succession Planning:**
- Communities encouraged to maintain multiple Admins for redundancy
- Admins manage database replication to their own nodes
- If Admin leaves community, use `AdminChange` proposal to remove role

**Failover Coordination:**
- Not technically enforced (per ADR-001: manual failover)
- Community coordinates via alternate communication channel
- Admin with most recent database replica spins up new primary

### 8. User Interface

**Proposal List Page:**
- Tabs: Active, Passed, Rejected, All
- Each proposal shows:
  - Title, creator, type, voting deadline
  - Current vote count or percentage (if transparent)
  - "Vote" button (if active and not yet voted)

**Proposal Detail Page:**
- Full description (Markdown rendered)
- Voting configuration displayed
- Vote casting interface
- Real-time vote tally
- Discussion thread (comments on proposal)
- If transparent: List of votes cast

**Create Proposal Page:**
- Form with all configuration options
- Presets for common proposal types
- Preview before publishing

**My Votes Page:**
- List of all proposals member has voted on
- Quick reference for participation history

### 9. Anti-Manipulation Measures

**Sybil Resistance (Multiple Fake Accounts):**
- **Invite code system (ADR-004):** Admin controls who joins
- **Physical proximity:** Members know each other, fake accounts obvious
- **Community size:** Small scale makes sockpuppets impractical

**Vote Buying/Coercion:**
- **Anonymous voting option:** Prevents verification of how someone voted
- **Community trust:** Physical proximity enables in-person verification if disputes arise

**Proposal Spam:**
- Optional: Require moderator approval for proposals
- Optional: Limit proposals per member per time period
- Community governance settings control this

**Admin Override:**
- **No admin override:** Once proposal passes, outcome is enforced automatically
- Admins cannot reverse democratic decisions
- Exception: Database-level rollback in case of bugs (logged in audit trail)

## Consequences

### Positive Consequences

- **Democratic legitimacy:** Community decisions are transparent and enforceable
- **Automated enforcement:** Role changes happen automatically based on votes (no admin discretion)
- **Flexibility:** Each community can establish its own governance norms
- **Transparency:** Voting records provide accountability
- **Member empowerment:** Any member can propose changes
- **Special proposal types:** Handles role changes, bans, configuration without admin intervention
- **Future-proof:** Can add new proposal types (platform configuration, moderation policies, etc.)
- **Aligned with mission:** Technical infrastructure enforces democratic principles
- **Simple for users:** Familiar voting interface, clear outcomes

### Negative Consequences

- **Complexity:** Proposal system adds significant feature scope
- **Governance learning curve:** Communities must learn to use democratic tools effectively
- **Minority rights:** Simple majority can override minority preferences (mitigated by configurable thresholds)
- **Participation burden:** Requires member engagement to function well
- **Decision speed:** Voting periods delay urgent decisions (mitigated by short voting periods for emergencies)
- **Bikeshedding risk:** Trivial decisions may consume disproportionate time
- **Database growth:** Proposals and votes add data over time (acceptable at small scale)

### Neutral Consequences

- **Governance evolution:** Communities develop norms through practice
- **Cultural variability:** Different neighborhoods will govern differently
- **Documentation needs:** Must document proposal best practices
- **Admin role implications:** Being Admin means managing infrastructure, not wielding power
- **Dispute resolution:** Communities need out-of-band mechanisms for vote disputes

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (proposals and votes replicate with database)
- ADR-002: Database Technology Selection (stored in SQLite)
- ADR-003: Application Framework and Technology Stack (Blazor UI for voting)
- ADR-004: Authentication and Authorization Model (roles changed via proposals)
- Future ADR: Content Moderation and Community Guidelines (moderation policies established via proposals)

### Implementation Phases

**Phase 1: Basic Proposals (MVP)**
- General proposals (Yes/No voting only)
- Transparent voting
- Manual outcome recording (no automated enforcement)
- Proposal list and detail pages

**Phase 2: Special Proposal Types**
- ModeratorChange and AdminChange proposals
- Automated role assignment on vote outcome
- MemberBan proposals

**Phase 3: Advanced Voting**
- Yes/No/Abstain voting
- Private (anonymous) voting option
- Minimum participation thresholds
- Multiple choice voting

**Phase 4: Governance Configuration**
- Admin UI for governance settings
- Community-specific default configurations
- Proposal approval workflows (optional)

**Phase 5: Enhanced Features**
- Ranked choice voting
- Proposal amendments
- Proxy voting (delegate vote to another member)
- Threaded discussions on proposals
- Email notifications for proposal activity

### References
- [Voting System Design Principles](https://en.wikipedia.org/wiki/Voting_system)
- [Platform Cooperativism](https://platform.coop/)
- [Liquid Democracy](https://en.wikipedia.org/wiki/Liquid_democracy) (potential future enhancement)
- [Quadratic Voting](https://en.wikipedia.org/wiki/Quadratic_voting) (potential future enhancement)

### Governance Best Practices (Documentation for Communities)

**Recommended Proposal Practices:**
1. **Discuss before proposing:** Use general discussion threads to gauge support
2. **Clear descriptions:** Explain context, rationale, and expected outcomes
3. **Reasonable voting periods:** 3-7 days for most decisions, longer for major changes
4. **Appropriate thresholds:** 50% for routine decisions, 66% for significant changes, 75%+ for constitutional changes
5. **Minimum participation:** Consider requiring 50%+ turnout for important decisions
6. **Transparency:** Default to public voting unless privacy is specifically needed

**Community Norms to Establish:**
- What types of decisions require proposals vs. moderator discretion?
- How often should major governance settings be reviewed?
- How to handle urgent decisions (e.g., spam attacks)?
- What constitutes appropriate proposal topics?
- How to handle proposal disputes or vote irregularities?

### Future Enhancements
- **Proposal templates:** Pre-filled forms for common proposal types
- **Proposal amendments:** Modify proposal based on community feedback before voting
- **Delegated voting:** Proxy vote to trusted community member
- **Ranked choice voting:** Multiple options with preference ordering
- **Quadratic voting:** Allocate voting power across multiple proposals
- **Proposal discussion threads:** Inline comments and deliberation
- **Email/SMS notifications:** Alert members about new proposals and voting deadlines
- **Proposal history:** Track how specific topics have evolved over time
- **Constitutional proposals:** Special category for fundamental governance changes (higher thresholds)