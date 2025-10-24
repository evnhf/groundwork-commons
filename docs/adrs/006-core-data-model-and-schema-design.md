# 006. Core Data Model and Schema Design

## Status

Accepted

## Context

Groundwork Commons needs a content model that supports diverse neighborhood interactions while maintaining simplicity and flexibility. Unlike traditional social platforms optimized for viral engagement, we're building infrastructure for practical community needs: sharing information, coordinating resources, asking for help, making decisions, and facilitating local exchange.

### Problem Statement

We need to define:
- The core data entities and their relationships
- How different types of community content are modeled (shares, threads, marketplace, proposals, etc.)
- The schema design that balances flexibility with structure
- How content is organized, discovered, and displayed
- Interaction patterns (comments, reactions, mentions, editing)

### Design Principles

1. **Unified Data Model:** Different content types (shares, threads, marketplace, proposals) share common structure with type-specific metadata
2. **Practical Interactions:** Support real neighborhood needs (buy/sell, ask/offer, discussions, decisions)
3. **Positive Engagement:** Avoid addictive mechanics (no simple "likes"); focus on meaningful interaction
4. **Flexibility:** Easy to add new content types without schema migrations
5. **Simplicity:** Small scale (5-50 users) allows simpler patterns

### Unique Requirements

- **Multi-purpose content:** Same post might be a marketplace listing, a discussion starter, or a proposal
- **Metadata flexibility:** Buy/sell posts need price/condition; proposals need voting config; shares are simple
- **Feed aggregation:** Single feed shows all activity across content types
- **Convertibility:** Shares can become threads, threads can become proposals
- **Community matching:** System helps pair asks with offers

### Alternatives Considered

**Separate Tables Per Content Type**
- Create distinct tables: `Shares`, `Threads`, `MarketplaceListings`, `Proposals`, etc.
- Pros: Strong typing, explicit schema per type
- Cons: Difficult to query unified feed, rigid structure, harder to convert between types
- Rejected: Too rigid for flexible community needs

**Key-Value Store for Posts**
- Store all posts as JSON blobs with no structured schema
- Pros: Ultimate flexibility
- Cons: No referential integrity, difficult to query efficiently, harder to migrate data
- Rejected: Loses relational database benefits

**Polymorphic Association with Shared Base**
- `Posts` table with `PostType` enum and type-specific metadata in JSON columns
- Pros: Single source of truth, easy feed queries, flexible metadata per type
- Pros: Can convert between types by changing `PostType` and metadata
- Chosen: Best balance of structure and flexibility

**Separate Schema, Unified Feed Table**
- Different tables for content, but a `Feed` table that references all types
- Pros: Strong typing per content type
- Cons: Complex feed queries, duplication, synchronization issues
- Rejected: Complexity outweighs benefits at small scale

## Decision

We will use a **unified polymorphic content model** where all community content (shares, threads, marketplace, asks/offers, proposals) is stored in a single `Posts` table with a `PostType` discriminator and flexible JSON metadata for type-specific fields.

### 1. Core Entity: Post

**The Post Model:**

All content is fundamentally a "Post" with common fields and type-specific metadata.

**Key Fields:**
- **Identity:** Unique ID, author reference
- **Content Type:** Discriminator field (`PostType`) determines how the post is displayed and what metadata is available
- **Common Content:** Title (optional), content (markdown), media URLs (JSON array), link data
- **Type-Specific Metadata:** JSON field storing fields specific to each post type (prices, categories, voting config, etc.)
- **Lifecycle Tracking:** Created/updated timestamps, edit flags, soft delete support
- **Status:** Lifecycle state (Active, Sold, Fulfilled, Closed, Expired, Archived)
- **Engagement Metrics:** Denormalized counts for comments and reactions, last activity timestamp for feed sorting
- **Relationships:** Parent post reference (for content conversion tracking), collections of comments and reactions

**Post Types:**
- `Share` - Simple status update, image share, link share
- `Thread` - Long-form discussion with title
- `MarketplaceSell` - Item for sale
- `MarketplaceBuy` - Looking to buy (WTB)
- `Ask` - Request for help/service
- `Offer` - Offer help/service
- `Proposal` - Voting proposal (references ADR-005)

**Post Status Values:**
- `Active` - Default for most content
- `Closed` - Thread/discussion closed to new comments
- `Sold` - Marketplace item sold
- `Fulfilled` - Ask/offer fulfilled
- `Expired` - Time-limited content expired
- `Archived` - Preserved but hidden from feed

### 2. Type-Specific Metadata (JSON)

Each `PostType` stores additional fields in the `TypeMetadata` JSON column:

**Share (Simple Post):**
```json
{
  "mood": "informational" // Optional: celebratory, urgent, question, etc.
}
```

**Thread (Discussion):**
```json
{
  "isPinned": false,
  "isLocked": false // Moderators can lock to prevent new comments
}
```

**MarketplaceSell:**
```json
{
  "price": 50.00,
  "currency": "USD",
  "condition": "Like New", // New, Like New, Good, Fair
  "category": "Electronics",
  "location": "123 Main St" // Optional pickup location
}
```

**MarketplaceBuy (Want to Buy):**
```json
{
  "maxPrice": 100.00,
  "category": "Furniture",
  "urgency": "Flexible" // ASAP, Flexible
}
```

**Ask (Need Help):**
```json
{
  "category": "Technical Help", // Handyman, Technical Help, Pet Care, etc.
  "urgency": "Not Urgent", // Urgent, Soon, Not Urgent
  "offeredCompensation": "Will pay $20/hour",
  "estimatedDuration": "2 hours"
}
```

**Offer (Providing Service):**
```json
{
  "category": "Technical Help",
  "availability": "Weekends",
  "isPaid": false, // Free or paid service
  "hourlyRate": null
}
```

**Proposal:**
```json
{
  // References Proposal table from ADR-005
  "proposalId": 42,

  // Quick summary for feed display
  "votingEndsAt": "2026-01-15T00:00:00Z",
  "currentVoteCount": 12,
  "passingThreshold": 0.66
}
```

### 3. Comments Model

Comments are replies to posts (and replies to other comments for nested threading).

**Key Fields:**
- **Identity:** Unique ID, references to post and author
- **Content:** Markdown text, optional media URLs (JSON array)
- **Threading:** Parent comment reference (null for top-level), depth indicator, collection of child replies
- **Lifecycle:** Created/updated timestamps, edit and soft delete flags
- **Engagement:** Collection of reactions

**Comment Threading:**
- Comments can nest up to 3 levels deep (prevents excessive nesting)
- Depth 0: Top-level comment on post
- Depth 1: Reply to comment
- Depth 2: Reply to reply
- Depth 3+: Flatten to avoid UI complexity

### 4. Reactions Model

Instead of simple "likes," we support expressive emoji reactions.

**Key Fields:**
- **Identity:** Unique ID
- **Polymorphic Target:** References either a post or comment (exactly one must be set)
- **User and Emoji:** User reference and unicode emoji string (e.g., "👍", "❤️", "🤔")
- **Timestamp:** When the reaction was created

**Reaction Rules:**
- One reaction per user per post/comment (can change reaction)
- No reaction counters displayed prominently (avoids engagement metric focus)
- Reactions visible on hover/click (less prominent than traditional likes)
- Curated emoji set to encourage positive, constructive reactions

**Avoiding Negativity:**
- No "dislike" or thumbs down emoji
- Focus on: ❤️ (support), 😊 (happy), 🤔 (thinking), 🙏 (thank you), 👏 (appreciate), 🎉 (celebrate)

### 5. Mentions Model

Track @mentions for notifications.

**Key Fields:**
- **Identity:** Unique ID
- **Location:** References either a post or comment where the mention occurred (exactly one must be set)
- **Mentioned User:** Reference to the user who was mentioned
- **Mentioning User:** Reference to the user who created the mention
- **Notification State:** Timestamp and read/unread flag for notification tracking

### 6. Feed Model (View)

The **Feed** is not a separate table but a **query/view** that aggregates all posts.

**Feed Item Components:**
- Post data
- Item type indicator (NewPost, NewComment, ProposalUpdate, etc.)
- Activity timestamp for sorting

**Feed Sorting Algorithms:**

**Chronological (Default):**
- Query all non-deleted posts with Active status
- Sort by creation timestamp (newest first)

**Activity-Based (Most Engaged):**
- Query all non-deleted posts with Active status
- Sort by last activity timestamp (most recently active first)

**Feed Filtering:**
- Filter by `PostType` (e.g., only show Marketplace)
- Filter by `Status` (e.g., hide Sold items)
- Search by keywords in `Title` or `Content`

### 7. Content Type Sections

Different views into the unified `Posts` table:

**1. Feed (Homepage):**
- Shows all `PostType` values
- Sorted by chronological or activity
- Includes: Shares, new Threads, Proposals, Marketplace highlights

**2. Buy/Sell (Marketplace):**
- Filter: `PostType IN (MarketplaceSell, MarketplaceBuy)`
- Show only `Status = Active` by default
- Option to toggle "Show Sold/Fulfilled"
- Category filtering via `TypeMetadata.category`

**3. Ask/Offer (Community Exchange):**
- Filter: `PostType IN (Ask, Offer)`
- **Smart Matching:** Algorithm suggests relevant Offers for Asks (match by category, keywords)
- Show active asks/offers prominently
- Mark as fulfilled when matched

**4. Threads (Discussions):**
- Filter: `PostType = Thread`
- Sorted by activity (recent comments bump to top)
- Pinned threads appear first

**5. Proposals (Governance):**
- Filter: `PostType = Proposal`
- Sorted by voting deadline (urgent votes first)
- Show active, then recently decided

### 8. Content Conversion

Posts can evolve from one type to another:

**Share → Thread:**
- User clicks "Convert to Thread" on a Share
- Create new `Post` with `PostType = Thread`
- Set `ParentPostId` to original Share
- Copy content, comments migrate

**Thread → Proposal:**
- User clicks "Create Proposal from Thread"
- Create new `Proposal` (ADR-005 model)
- Create new `Post` with `PostType = Proposal`
- Set `ParentPostId` to original Thread
- Link via `TypeMetadata.proposalId`

**Conversion Tracking:**
- Original post remains (marked as `Status = Archived`)
- New post references via `ParentPostId`
- UI shows lineage: "Originally posted as Share by Alice"

### 9. Database Schema (SQLite)

**Tables:**

```sql
-- Posts (unified content)
CREATE TABLE Posts (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    AuthorId TEXT NOT NULL,
    Type TEXT NOT NULL, -- Share, Thread, MarketplaceSell, etc.
    Title TEXT,
    Content TEXT NOT NULL,
    MediaUrls TEXT, -- JSON array
    LinkUrl TEXT,
    LinkPreviewData TEXT, -- JSON
    TypeMetadata TEXT, -- JSON
    CreatedAt INTEGER NOT NULL,
    UpdatedAt INTEGER,
    IsEdited INTEGER DEFAULT 0,
    IsDeleted INTEGER DEFAULT 0,
    DeletedAt INTEGER,
    DeletedByUserId TEXT,
    Status TEXT DEFAULT 'Active',
    CommentCount INTEGER DEFAULT 0,
    ReactionCount INTEGER DEFAULT 0,
    LastActivityAt INTEGER NOT NULL,
    ParentPostId INTEGER,
    FOREIGN KEY (AuthorId) REFERENCES AspNetUsers(Id),
    FOREIGN KEY (ParentPostId) REFERENCES Posts(Id)
);

CREATE INDEX idx_posts_type ON Posts(Type);
CREATE INDEX idx_posts_status ON Posts(Status);
CREATE INDEX idx_posts_activity ON Posts(LastActivityAt DESC);
CREATE INDEX idx_posts_author ON Posts(AuthorId);

-- Comments
CREATE TABLE Comments (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    PostId INTEGER NOT NULL,
    AuthorId TEXT NOT NULL,
    Content TEXT NOT NULL,
    MediaUrls TEXT, -- JSON array
    ParentCommentId INTEGER,
    Depth INTEGER DEFAULT 0,
    CreatedAt INTEGER NOT NULL,
    UpdatedAt INTEGER,
    IsEdited INTEGER DEFAULT 0,
    IsDeleted INTEGER DEFAULT 0,
    DeletedAt INTEGER,
    FOREIGN KEY (PostId) REFERENCES Posts(Id) ON DELETE CASCADE,
    FOREIGN KEY (AuthorId) REFERENCES AspNetUsers(Id),
    FOREIGN KEY (ParentCommentId) REFERENCES Comments(Id)
);

CREATE INDEX idx_comments_post ON Comments(PostId);
CREATE INDEX idx_comments_author ON Comments(AuthorId);
CREATE INDEX idx_comments_parent ON Comments(ParentCommentId);

-- Reactions
CREATE TABLE Reactions (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    PostId INTEGER,
    CommentId INTEGER,
    UserId TEXT NOT NULL,
    Emoji TEXT NOT NULL,
    CreatedAt INTEGER NOT NULL,
    FOREIGN KEY (PostId) REFERENCES Posts(Id) ON DELETE CASCADE,
    FOREIGN KEY (CommentId) REFERENCES Comments(Id) ON DELETE CASCADE,
    FOREIGN KEY (UserId) REFERENCES AspNetUsers(Id),
    UNIQUE(PostId, UserId), -- One reaction per user per post
    UNIQUE(CommentId, UserId), -- One reaction per user per comment
    CHECK ((PostId IS NOT NULL AND CommentId IS NULL) OR
           (PostId IS NULL AND CommentId IS NOT NULL))
);

CREATE INDEX idx_reactions_post ON Reactions(PostId);
CREATE INDEX idx_reactions_comment ON Reactions(CommentId);
CREATE INDEX idx_reactions_user ON Reactions(UserId);

-- Mentions
CREATE TABLE Mentions (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    PostId INTEGER,
    CommentId INTEGER,
    MentionedUserId TEXT NOT NULL,
    MentionedByUserId TEXT NOT NULL,
    CreatedAt INTEGER NOT NULL,
    IsRead INTEGER DEFAULT 0,
    FOREIGN KEY (PostId) REFERENCES Posts(Id) ON DELETE CASCADE,
    FOREIGN KEY (CommentId) REFERENCES Comments(Id) ON DELETE CASCADE,
    FOREIGN KEY (MentionedUserId) REFERENCES AspNetUsers(Id),
    FOREIGN KEY (MentionedByUserId) REFERENCES AspNetUsers(Id),
    CHECK ((PostId IS NOT NULL AND CommentId IS NULL) OR
           (PostId IS NULL AND CommentId IS NOT NULL))
);

CREATE INDEX idx_mentions_user ON Mentions(MentionedUserId);
CREATE INDEX idx_mentions_read ON Mentions(IsRead);
```

### 10. Denormalization for Performance

Given small scale (5-50 users), we denormalize certain counts:

**Post.CommentCount:**
- Incremented when comment is added
- Decremented when comment is deleted
- Avoids `COUNT(*)` queries on feed display

**Post.ReactionCount:**
- Sum of all reactions on post
- Updated when reactions added/removed

**Post.LastActivityAt:**
- Updated when: New comment added, post edited, reaction added
- Used for activity-based feed sorting

**Trade-off:** Slightly more complex write operations, but much faster feed queries.

### 11. Ask/Offer Matching Algorithm

**Goal:** Help neighbors find each other for mutual aid.

**Matching Logic:**

For a given Ask post:
1. Parse the Ask's metadata to extract category and keywords
2. Query all active Offer posts
3. For each Offer, check if it matches:
   - **Category match:** Offer category equals Ask category
   - **Keyword match:** Significant keyword overlap between Ask and Offer content
4. Return list of matching Offers, sorted by relevance

**UI Integration:**
- When viewing an Ask, show "Matching Offers" sidebar
- When viewing an Offer, show "Relevant Asks" sidebar
- Notification: "Your offer matches a new ask!"

## Consequences

### Positive Consequences

- **Unified data model:** Single source of truth for all content, easy to query feed
- **Flexible schema:** JSON metadata allows type-specific fields without migrations
- **Easy conversions:** Shares can become Threads, Threads can become Proposals
- **Practical features:** Buy/Sell, Ask/Offer address real neighborhood needs
- **Positive engagement:** Emoji reactions replace addictive "likes"
- **Nested discussions:** Comments support meaningful conversation
- **Activity tracking:** Feed can surface engaged discussions
- **Scalable:** Schema handles 5-50 users efficiently, could scale to hundreds
- **Simple queries:** SQLite + indexes handle all queries at this scale

### Negative Consequences

- **Schema flexibility cost:** JSON metadata harder to query than structured columns (acceptable trade-off)
- **Denormalization complexity:** CommentCount/ReactionCount require careful update logic
- **Type safety:** TypeMetadata is JSON, not strongly typed (mitigated by C# models)
- **No full-text search initially:** SQLite FTS extension needed for advanced search (future enhancement)
- **Comment depth limit:** Prevents deep nesting (acceptable UX trade-off)
- **Migration risk:** Changing PostType enum requires data migration

### Neutral Consequences

- **Feed algorithm tuning:** Will need experimentation to find right balance (chronological vs. activity)
- **Ask/Offer matching:** Simple algorithm initially, can enhance with better matching later
- **Reaction emoji curation:** Need to choose meaningful emoji set
- **Content moderation:** Unified model means one set of moderation rules applies to all types
- **Storage growth:** MediaUrls and LinkPreviewData grow database (mitigated by small scale)

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (all content replicates via Litestream)
- ADR-002: Database Technology Selection (SQLite with JSON1 extension)
- ADR-003: Application Framework and Technology Stack (EF Core models)
- ADR-004: Authentication and Authorization Model (Author relationships)
- ADR-005: Democratic Governance and Proposal System (Proposal PostType integration)
- ADR-007: Content Features and Capabilities (will detail rich media, editing, etc.)

### Implementation Phases

**Phase 1: Core Content Model (MVP)**
- Posts table with Share and Thread types
- Comments (flat, single-level)
- Basic feed (chronological)
- Text-only content

**Phase 2: Marketplace**
- MarketplaceSell and MarketplaceBuy post types
- TypeMetadata for pricing, condition, category
- Marketplace filtered view
- Status updates (Active → Sold)

**Phase 3: Ask/Offer System**
- Ask and Offer post types
- Matching algorithm (simple category matching)
- Fulfillment workflow
- Notifications for matches

**Phase 4: Rich Interactions**
- Nested comments (3 levels deep)
- Emoji reactions
- @mentions and notifications
- Activity-based feed sorting

**Phase 5: Content Conversion**
- Share → Thread conversion
- Thread → Proposal conversion
- Lineage tracking

**Phase 6: Enhancements**
- Rich media (images, file uploads)
- Link previews
- Full-text search (SQLite FTS5)
- Advanced Ask/Offer matching (keyword, ML-based)

### Common Operations

**Creating a Share:**
1. Create new Post entity with author, PostType.Share, content, and media URLs
2. Set timestamps (created, last activity)
3. Set status to Active
4. Save to database

**Creating a Marketplace Listing:**
1. Create new Post entity with author, PostType.MarketplaceSell, title, and content
2. Populate TypeMetadata JSON with price, currency, condition, category, location
3. Add media URLs for product images
4. Set status to Active
5. Save to database

**Creating an Ask:**
1. Create new Post entity with author, PostType.Ask, title, and content
2. Populate TypeMetadata JSON with category, urgency, compensation, estimated duration
3. Set status to Active
4. Save to database
5. Trigger matching algorithm to find relevant Offers
6. Send notifications to matching Offer authors

**Adding a Comment with Mention:**
1. Create new Comment entity linked to post and author
2. Parse content for @mentions (username patterns)
3. For each mentioned user:
   - Lookup user by username
   - Create Mention entity linking comment, mentioned user, and mentioning user
   - Mark mention as unread for notification
4. Update parent post's last activity timestamp and increment comment count
5. Save all entities to database

**Querying the Feed (Chronological):**
1. Filter posts: not deleted AND status is Active
2. Include author data for display
3. Order by creation timestamp (descending)
4. Limit to reasonable page size (e.g., 50 posts)

**Querying the Feed (Activity-Based):**
1. Filter posts: not deleted AND status is Active
2. Include author data for display
3. Order by last activity timestamp (descending)
4. Limit to reasonable page size (e.g., 50 posts)

**Querying Marketplace:**
1. Filter posts: type is MarketplaceSell or MarketplaceBuy
2. Filter: not deleted AND status is Active
3. Include author data for display
4. Order by creation timestamp (descending)

### TypeMetadata Structure Examples

See section 2 above for detailed TypeMetadata JSON examples for each post type (MarketplaceSell, MarketplaceBuy, Ask, Offer, Thread, Proposal).

### References
- [Activity Streams](https://www.w3.org/TR/activitystreams-core/) - W3C standard for social activity
- [JSON in SQLite](https://www.sqlite.org/json1.html) - SQLite JSON1 extension docs
- [Polymorphic Associations in EF Core](https://learn.microsoft.com/en-us/ef/core/modeling/inheritance)
- [Feed Ranking Algorithms](https://en.wikipedia.org/wiki/News_Feed) - Background on feed design

### Future Enhancements
- **Full-text search:** SQLite FTS5 for searching posts/comments
- **Advanced Ask/Offer matching:** ML-based keyword extraction, semantic matching
- **Content recommendations:** "You might be interested in this Ask based on your Offers"
- **Saved posts:** Members can bookmark posts for later
- **Post scheduling:** Schedule posts for future publication
- **Draft posts:** Save posts as drafts before publishing
- **Post templates:** Common templates for marketplace, asks, proposals
- **Content analytics:** Track engagement patterns (privacy-preserving)
- **Hashtags:** Optional tagging system for categorization
- **Follow threads:** Subscribe to specific thread updates