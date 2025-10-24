# 007. Content Features and Capabilities

## Status

Accepted

## Context

Groundwork Commons needs robust content handling capabilities that support meaningful community interactions while respecting the constraints of a small-scale, self-hosted platform. Unlike enterprise platforms with unlimited storage and CDN infrastructure, our content system must balance feature richness with operational simplicity for neighborhoods running their own infrastructure.

### Problem Statement

We need to define:
- How rich content (formatting, images, files) is created and stored
- Content validation rules and size limits
- Media handling (upload, storage, delivery, replication)
- Link preview generation and caching strategy
- Edit functionality and restrictions by content type
- Content lifecycle and moderation approach

### Design Principles

1. **Practical Richness:** Support common content needs (formatting, images, files) without bloating storage
2. **Operational Simplicity:** Avoid complex CDN/image processing pipelines that burden small operators
3. **Data Sovereignty:** All content stored locally, no external dependencies required
4. **Resilience:** Content replicates with database to survive node failures
5. **Performance:** Small scale (5-50 users) allows simpler patterns than viral platforms
6. **Trust-Based:** Neighbors sharing physical space reduces need for aggressive content filtering

### Unique Requirements

- **Self-hosted media:** All media stored locally (filesystem or cloud, operator's choice)
- **Configurable storage:** Operators control media limits and storage backends
- **Simple processing:** Avoid complex image manipulation in MVP (client-side resizing instead)
- **Offline capability:** Link previews cached in-memory, no external API dependencies required
- **Edit transparency:** Community scale means edits should be trackable
- **Proposal immutability:** Special handling to prevent editing proposals during voting

### Alternatives Considered

**External Media Storage (Required)**
- Force all operators to use cloud object storage (S3, Azure Blob)
- Pros: Unlimited scale, fast delivery, offloads bandwidth
- Cons: External dependency, recurring costs, contradicts data sovereignty
- Rejected: Incompatible with community ownership and offline-first principles

**Markdown-Only (No Rich Media)**
- Support only Markdown text, no images or files
- Pros: Simplest implementation, minimal storage
- Cons: Too limiting for practical community needs (sharing photos, documents)
- Rejected: Insufficient for real-world neighborhood use cases

**Database BLOB Storage**
- Store media as BLOBs in SQLite database
- Pros: Single storage location, replicates automatically with Litestream
- Cons: Database bloat, poor performance for large files, difficult to serve efficiently
- Rejected: SQLite performance degrades with large BLOBs

**Full Rich Text Editor (WYSIWYG)**
- Provide rich text editor like Quill, TinyMCE
- Pros: Familiar user experience for non-technical users
- Cons: Complex HTML sanitization, security risks, accessibility issues
- Rejected: Markdown provides better security and simplicity

**Chosen: Markdown + Filesystem/Cloud Media Storage**
- Use Markdown for text formatting with GitHub-flavored extensions
- Store media files on filesystem or S3-compatible storage (operator's choice)
- Browser-side image resizing/conversion before upload
- In-memory link preview caching
- Configurable edit rules with special proposal handling
- Pros: Balance of richness, simplicity, security, sovereignty, and flexibility
- Chosen: Best fit for hyperlocal self-hosted platform

## Decision

We will implement a **Markdown-based content system** with **flexible media storage** (filesystem or S3-compatible cloud), **browser-side image processing**, **in-memory link preview caching**, and **configurable edit policies** optimized for small-scale community infrastructure.

### 1. Text Formatting: Markdown

**Markdown Support:**

All text content (posts, comments) uses **GitHub-Flavored Markdown (GFM)** for formatting:

**Standard Markdown:**
- Headers (`# H1`, `## H2`, etc.) - available in Threads and Proposals, not Shares
- Bold (`**bold**`), italic (`*italic*`)
- Links (`[text](url)`)
- Lists (ordered and unordered)
- Blockquotes (`> quote`)
- Code blocks (` ``` ` for multi-line, `` ` `` for inline)

**GitHub-Flavored Extensions:**
- Tables (better spacing and structure)
- Strikethrough (`~~text~~`)
- Task lists (`- [ ] todo`, `- [x] done`)
- Autolinks (URLs become clickable without explicit Markdown syntax)

**Content Type Restrictions:**
- **Shares:** Bold, italic, links only (no headlines)
- **Threads/Proposals:** Full Markdown including headlines
- **Comments:** Full Markdown except headlines

**Parser Implementation:**
- Server-side rendering to HTML for display
- Client-side preview for content creation
- Sanitization to prevent XSS attacks (no raw HTML allowed)

**Security:**
- No raw HTML permitted in Markdown
- Sanitize all rendered output
- Disable dangerous extensions (script tags, iframes)

**Content Limits:**
- Configurable per instance for database flooding prevention
- Suggested defaults: 10,000 characters for posts, 2,000 for comments
- No hard limits enforced by design (operators decide)

### 2. Media Upload and Storage

**Supported Media Types:**

**Images:**
- Formats: JPEG, PNG, WebP, GIF
- Maximum file size: Configurable per instance
- Maximum images per post: Configurable (default: 10)
- **Browser-side processing:**
  - Images converted to JPEG before upload
  - Resized if needed (quality ~75%, preserve aspect ratio)
  - No max dimensions enforced (rendered smaller in UI as needed)

**File Attachments:**
- All file types supported (no whitelist in MVP)
- Maximum file size: Configurable per instance (suggested default: 10 MB)
- Previews shown when possible based on metadata
- Supported on: Proposals, Threads (exposed for Posts if users demand)

**Comments Media:**
- Support up to 1 image per comment
- Same processing as post images

**Storage Architecture:**

Operators choose storage backend via configuration:

**Option 1: Filesystem Storage (Default)**
```
/var/groundwork-commons/
├── data/
│   └── groundwork.db           # SQLite database
├── media/
│   ├── posts/{PostId}/
│   │   ├── {filename}.jpg
│   │   └── {filename}.pdf
│   └── profiles/{UserId}/
│       └── avatar.jpg
└── backups/                    # Litestream replicas
```

- Media stored alongside database on same filesystem
- Replicates with Litestream (see ADR-001 considerations)
- Simple deployment, no external dependencies

**Option 2: Cloud Storage (S3-Compatible)**
- Any S3-compatible storage provider (AWS S3, Azure Blob Storage, Cloudflare R2, MinIO, etc.)
- Configured via environment variables (bucket, credentials, endpoint)
- Cloud storage typically handles its own redundancy
- Reduces local storage requirements

**Upload Flow:**
1. User selects file via upload dialog
2. **Browser processes image:** Resize/convert to JPEG at ~75% quality
3. Client requests upload URL from API: `POST /api/media/upload-url`
4. Server validates request, generates signed upload URL (local or cloud)
5. Client uploads file directly to storage endpoint
6. Storage returns file URL
7. Client includes URL in post submission
8. Server validates and saves post with media URLs

**Media URL Storage:**
- Stored in `Posts.MediaUrls` and `Comments.MediaUrls` JSON arrays
- URLs are absolute paths (local: `/media/posts/123/abc.jpg`, cloud: `https://...`)
- Frontend doesn't care about storage backend (URL abstraction)

**Deletion Handling:**
- Deleting a post/comment triggers media cleanup
- Background job removes orphaned media files
- Configurable retention period for soft-deleted content

### 3. Media Replication Strategy

**Filesystem Storage + Litestream:**

Per ADR-001, Litestream handles database replication. For media files:

**If Litestream supports filesystem replication:**
- Configure Litestream to replicate entire `/var/groundwork-commons/` directory
- Media and database replicate together to same targets
- Restoration restores both database and media atomically

**If Litestream does not support filesystem replication:**
- Need separate mechanism for media redundancy:
  - **Option A:** Scheduled rsync/rclone to backup locations
  - **Option B:** Application-level media sync to peer nodes
  - **Option C:** Rely on cloud storage redundancy (if using cloud backend)
- This will be determined during implementation based on Litestream capabilities

**Cloud Storage:**
- Cloud providers handle their own redundancy (S3 durability: 99.999999999%)
- No additional replication mechanism needed
- Primary node failure doesn't lose media (stored externally)

**Hybrid Approach:**
- Operators can use cloud storage as Litestream replication target
- Database replicates to cloud, media stored in cloud
- Single cloud location contains full backup

### 4. Link Previews

**Goal:** Display rich previews for shared URLs (like social media link cards).

**Caching Strategy:**

**In-Memory Cache Only:**
- Link preview data cached in server memory (not database)
- Cache duration: 24 hours (configurable)
- Cache eviction: LRU or simple time-based expiration
- On server restart: Cache cleared, previews re-fetched on demand

**Fetch Flow:**
1. User includes URL in post
2. On rendering client-side, UI requests preview: `GET /api/link-preview?url={url}`
3. Server checks in-memory cache first
4. If cached and fresh: Return cached preview data immediately
5. If not cached or expired:
   - Fetch HTML from URL (timeout: 5 seconds)
   - Extract Open Graph tags or fallback to HTML meta tags
   - Cache preview data in memory
   - Return preview data to client
6. Client displays preview card

**Preview Data Structure:**
```json
{
  "url": "https://example.com/article",
  "title": "Article Title",
  "description": "Brief summary...",
  "imageUrl": "https://example.com/og-image.jpg",
  "siteName": "Example Site",
  "favicon": "https://example.com/favicon.ico"
}
```

**Error Handling:**
- If URL unreachable: Return minimal preview (URL only)
- Timeout after 5 seconds to avoid blocking
- Display generic link if preview fetch fails

**Trade-offs:**
- Rebuilds naturally for visible links after restart (acceptable)
- Saves database storage (preview data not persisted)
- Reduces Litestream replication overhead

### 5. Edit Functionality

**Edit Permissions:**

**Who Can Edit:**
- Authors can edit their own posts/comments
- Moderators can edit any post/comment (with audit trail)
- Admins can edit any content (with audit trail)

**Time Limits:**
- **No time limits** for editing posts or comments
- Users can edit anytime (except special cases below)

**Edit Restrictions by Content Type:**

**Posts (Shares, Threads, Ask/Offer, Buy/Sell):**
- Editable anytime by author
- No restrictions

**Proposals:**
- **Editable:** When proposal status is Draft or not yet "Open for Vote"
- **Not editable:** Once proposal enters "Open for Vote" status
- **Editable again:** If voting is closed prematurely without completion
- **Never editable:** Once proposal is marked as Resolved (successful or not)

**Rationale:** Prevents changing proposal content while community is actively voting on it.

**Edit Tracking:**

**Database Fields:**
- `IsEdited` (boolean): Flag indicating content was modified
- `UpdatedAt` (timestamp): When last edit occurred

**UI Indicators:**
- Display "(edited)" badge next to timestamp
- Clicking badge shows edit timestamp: "Last edited: Jan 10, 2026 at 3:45pm"

**No Edit History:**
- MVP does not store edit snapshots
- Only tracks that content was edited and when
- Acceptable for small trusted communities
- Future enhancement: Add `EditHistory` JSON column for full transparency

**Media Edit Rules:**
- Editing post does not allow adding/removing media after publication (MVP)
- Future: Allow media edits within grace period (e.g., 5 minutes)

### 6. Content Validation

**Server-Side Validation:**

**General Rules:**
- Markdown parsing validates syntax
- Reject if parsing fails
- Sanitize HTML output to remove dangerous tags

**Content Length:**
- Configurable per instance (operators set limits)
- No hard-coded limits enforced
- Database flooding prevention left to operator discretion

**Media Validation:**
- File size: Must be within operator-configured limits
- MIME type validation: Check actual file content (not just extension)
- Virus scanning: Optional integration (operator decides)

**Rate Limiting:**
- No built-in rate limiting in MVP
- Small scale (5-50 users) and trust model reduce abuse risk
- Operators can add reverse proxy rate limiting if needed

**Content Moderation:**
- No automated content filters (profanity, keywords)
- Manual moderation by community moderators
- Trust-based approach appropriate for physical neighbors

### 7. Browser-Side Image Processing

**Processing Steps:**

1. User selects image file(s) in upload dialog
2. Browser JavaScript reads file via FileReader API
3. Load image into canvas element
4. Calculate resize dimensions (preserve aspect ratio, no max dimensions)
5. Convert to JPEG format at ~75% quality
6. Generate blob from canvas
7. Upload processed JPEG to server

**Benefits:**
- Reduces server-side processing requirements
- Smaller file sizes uploaded (faster, less bandwidth)
- Operator doesn't need ImageSharp or similar libraries
- Works in all modern browsers

**Trade-offs:**
- Requires JavaScript enabled
- Original image metadata lost (EXIF data)
- Quality reduction may be noticeable for high-res photos
- Acceptable for neighborhood social sharing

### 8. Content Rendering

**Markdown to HTML:**

**Server-Side Rendering:**
- Parse Markdown to HTML on server
- Apply sanitization to prevent XSS
- Inject custom CSS classes for styling
- Return rendered HTML to client

**Client-Side Preview:**
- Blazor components render Markdown preview in real-time
- Shows "Preview" tab in content creation form
- Same parsing logic as server (consistency)

**Media Rendering:**
- Images: Display inline with responsive CSS (`max-width: 100%`)
- Files: Display as download links with file metadata
- Link previews: Render as preview cards with title, description, image

## Consequences

### Positive Consequences

- **Balanced richness:** Markdown provides formatting without HTML security risks
- **Storage flexibility:** Operators choose filesystem or cloud based on needs
- **Simple deployment:** No image processing libraries required (browser-side)
- **Edit transparency:** Community members see when content is modified
- **Proposal integrity:** Immutable proposals during voting ensure fair governance
- **Data sovereignty:** Filesystem storage option keeps all data local
- **Offline-first friendly:** Link previews work without external dependencies
- **Performance:** In-memory caching fast, small scale allows simple patterns
- **Configurable limits:** Operators control storage quotas and file sizes
- **Security:** Markdown sanitization prevents XSS attacks

### Negative Consequences

- **Learning curve:** Markdown less intuitive than WYSIWYG for non-technical users
- **Media replication complexity:** Filesystem storage requires separate replication mechanism if Litestream doesn't support it
- **Image quality loss:** Browser-side JPEG conversion reduces quality
- **No edit history:** MVP doesn't track edit snapshots (privacy/storage trade-off)
- **Link preview volatility:** In-memory cache lost on restart
- **No advanced image features:** No thumbnails, format conversion, or optimization (MVP)
- **Storage monitoring required:** Operators must manually monitor quotas

### Neutral Consequences

- **Markdown syntax:** Users may need brief tutorial (acceptable for engaged community)
- **Storage backend choice:** Operators must decide filesystem vs. cloud at setup
- **Client-side processing:** Requires modern browser with JavaScript enabled
- **Media URL management:** Changing storage backend requires database updates
- **Proposal edit logic:** More complex validation rules for proposal content
- **Rate limiting:** Deferred to reverse proxy or future enhancement

---

## Notes

### Related ADRs
- ADR-001: Data Replication Strategy (media replication considerations, may require update)
- ADR-002: Database Technology Selection (SQLite stores media URLs, not BLOBs)
- ADR-003: Application Framework and Technology Stack (Blazor components, API endpoints)
- ADR-005: Democratic Governance and Proposal System (proposal edit restrictions)
- ADR-006: Core Data Model and Schema Design (Posts.Content, Posts.MediaUrls, Comments.MediaUrls)

### Implementation Phases

**Phase 1: Markdown + Basic Media (MVP)**
- Implement GitHub-Flavored Markdown rendering
- Support browser-side image upload (JPEG conversion)
- Filesystem storage only
- Simple media URL storage in database
- Edit tracking (IsEdited flag only)

**Phase 2: Link Previews**
- Fetch Open Graph metadata
- In-memory caching with 24-hour expiration
- Display preview cards in UI
- Handle fetch errors gracefully

**Phase 3: Cloud Storage Support**
- Add S3-compatible storage backend option
- Configuration via environment variables
- Upload URL generation for cloud targets
- Media deletion for cloud-stored files

**Phase 4: Media Replication**
- Investigate Litestream filesystem replication capabilities
- Implement alternative replication mechanism if needed
- Document backup/restore procedures for media
- Test failover scenarios with media

**Phase 5: Enhanced Editing**
- Add edit history tracking (optional feature)
- Edit history viewer UI
- Configurable by operator
- Grace period for media edits

**Phase 6: Future Enhancements**
- Optional server-side image processing (thumbnails, optimization)
- File type validation and virus scanning
- Draft posts
- Scheduled publishing
- Advanced Markdown features (diagrams, math)

### Common Operations

**Uploading an Image:**
1. User selects image file in upload dialog
2. Browser resizes/converts to JPEG at 75% quality
3. Client requests upload URL: `POST /api/media/upload-url`
4. Server responds with signed URL (filesystem or cloud)
5. Client uploads JPEG to URL
6. Storage returns file URL
7. Client inserts Markdown: `![Description](/media/posts/42/abc123.jpg)`
8. User submits post with media URL included

**Fetching Link Preview:**
1. Client renders post with URL in link field
2. Client calls `GET /api/link-preview?url=https://example.com`
3. Server checks in-memory cache
4. If cached: Return cached data
5. If not cached: Fetch HTML, extract Open Graph tags, cache, return
6. Client displays preview card

**Editing a Proposal:**
1. Author clicks "Edit" on their proposal
2. Server checks proposal status
3. If "Open for Vote" or "Resolved": Reject edit (403 Forbidden)
4. If Draft or Closed: Allow edit
5. Author modifies content
6. Server validates, sets `IsEdited = true`, `UpdatedAt = now()`
7. Server saves proposal

**Deleting Post with Media:**
1. User deletes post (soft delete)
2. Server sets `IsDeleted = true`, `DeletedAt = now()`
3. Background job enqueues media cleanup task
4. After retention period: Job deletes media files from storage
5. Media URLs removed from orphaned file list

### Storage Sizing Estimates

For a 50-user community with active sharing:

**Assumptions:**
- Average 2 posts per user per week
- 50% of posts include 1 image (average 500 KB after JPEG conversion)
- 10% of posts include 1 file attachment (average 2 MB)

**Annual Storage:**
- Images: 50 users × 2 posts/week × 52 weeks × 50% × 500 KB = **1.3 GB/year**
- Files: 50 users × 2 posts/week × 52 weeks × 10% × 2 MB = **1.04 GB/year**
- Total: **~2.5 GB/year** (conservative estimate)

Operators should plan for **5-10 GB storage** for multi-year operation.

### Configuration Examples

**Filesystem Storage:**
```bash
STORAGE_BACKEND=filesystem
STORAGE_PATH=/var/groundwork-commons/media
MAX_IMAGE_SIZE_MB=5
MAX_FILE_SIZE_MB=10
MAX_IMAGES_PER_POST=10
```

**S3-Compatible Cloud Storage:**
```bash
STORAGE_BACKEND=s3
S3_BUCKET=my-community-media
S3_REGION=us-west-2
S3_ACCESS_KEY=xxx
S3_SECRET_KEY=xxx
S3_ENDPOINT=https://s3.amazonaws.com  # or MinIO, R2, etc.
MAX_IMAGE_SIZE_MB=5
MAX_FILE_SIZE_MB=10
MAX_IMAGES_PER_POST=10
```

### References
- [CommonMark Specification](https://commonmark.org/) - Markdown standard
- [GitHub-Flavored Markdown](https://github.github.com/gfm/) - GFM specification
- [Open Graph Protocol](https://ogp.me/) - Link preview metadata standard
- [Canvas API](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) - Browser image processing
- [AWS S3 API](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html) - S3-compatible storage
- [Litestream Documentation](https://litestream.io/) - Database replication

### Future Enhancements
- **Edit history tracking:** Store edit snapshots in JSON column
- **Version diffing:** Show side-by-side comparison of edits
- **Server-side image processing:** Optional thumbnails and optimization
- **Advanced media types:** Video upload and streaming
- **Voice notes:** Audio message recording and playback
- **Content templates:** Pre-filled forms for common post types
- **Media galleries:** Browse all community photos in grid view
- **Export/import:** Download all content and media for archival
- **Collaborative editing:** Operational transforms for real-time multi-user editing
- **Offline support:** Service workers for offline content creation with sync