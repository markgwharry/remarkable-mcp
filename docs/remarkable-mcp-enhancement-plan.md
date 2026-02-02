# remarkable-mcp Enhancement Plan

## Project Overview

This document outlines the plan to fork and extend [SamMorrowDrums/remarkable-mcp](https://github.com/SamMorrowDrums/remarkable-mcp) to add write operations and organisation capabilities for reMarkable tablets.

**Goal:** Create a comprehensive document organisation system that allows AI assistants (via MCP) to help users manage, reorganise, and tag their reMarkable library.

**Target Devices:** reMarkable 2, reMarkable Paper Pro Move

---

## Current State of remarkable-mcp

### Existing Capabilities ✅

| Tool | Purpose |
|------|---------|
| `remarkable_read` | Extract text from documents (typed text + OCR for handwriting) |
| `remarkable_browse` | Navigate folders, search by document name |
| `remarkable_search` | Content search across multiple documents |
| `remarkable_recent` | List recently modified documents |
| `remarkable_status` | Check connection health |
| `remarkable_image` | Get PNG/SVG images of pages |

### Key Features
- **SSH Mode:** Direct USB connection, 10-100x faster than cloud, no subscription required
- **Cloud Mode:** Works via reMarkable Cloud API (requires Connect subscription)
- **OCR Support:** Google Vision (best), Tesseract (fallback), or MCP Sampling
- **Native Text Extraction:** Type Folio and typed annotations extracted without OCR
- **Read-Only:** All current tools are read-only

### Technical Stack
- Python with FastMCP framework
- `rmscene` for .rm file parsing
- `PyMuPDF` for PDF extraction
- `Paramiko` for SSH transport

---

## Gap Analysis

### What's Missing for Organisation

| Need | Current State | Required |
|------|---------------|----------|
| Move documents | ❌ Not available | `remarkable_move()` |
| Create folders | ❌ Not available | `remarkable_create_folder()` |
| Rename items | ❌ Not available | `remarkable_rename()` |
| Delete items | ❌ Not available | `remarkable_delete()` |
| Tag management | ❌ Not exposed | `remarkable_tag()`, `remarkable_untag()` |
| Bulk operations | ❌ Not available | Batch move/tag functions |

### Open Issues in Upstream Repo

| Issue | Description | Status |
|-------|-------------|--------|
| #24 | Write support: Create documents and sync from Obsidian | Open, discussion |
| #26 | Enhanced search: Full-text indexing and semantic search | Open |
| #27 | Export features: PDF, Markdown, batch export | Open |
| #28 | Performance: Parallel registration, persistent cache | Open |
| #29 | Reliability: Auto-reconnection, retry logic | Open |

---

## reMarkable Filesystem Structure

Understanding the filesystem is critical for implementing write operations.

### Document Storage Location
```
/home/root/.local/share/remarkable/xochitl/
```

### File Structure per Document/Folder

Each item has a UUID and associated files:

```
{UUID}.metadata     # Core metadata (name, type, parent, deleted, pinned)
{UUID}.content      # Page info, tool settings, TAGS
{UUID}.local        # Sync state
{UUID}.pagedata     # Template info
{UUID}/             # Directory containing .rm files (pen strokes per page)
{UUID}.thumbnails/  # Thumbnail images
{UUID}.pdf          # Original PDF (if uploaded as PDF)
{UUID}.epub         # Original EPUB (if uploaded as EPUB)
{UUID}.tombstone    # Created when permanently deleted (firmware 3.5+)
```

### Key File: `{UUID}.metadata`

```json
{
  "visibleName": "Meeting Notes",
  "type": "DocumentType",        // or "CollectionType" for folders
  "parent": "abc123-def456...",  // UUID of parent folder, "" for root, "trash" for trash
  "deleted": false,
  "pinned": false,               // true = Favourite
  "lastModified": "1706789012345",
  "metadatamodified": "1706789012345",
  "modified": true,
  "synced": false,
  "version": 1
}
```

### Key File: `{UUID}.content`

Contains page information AND tags. Structure varies between documents and folders:
- **Folders:** Appears to contain a list of tags
- **Documents:** Contains page UUIDs, tool settings, and potentially tags

**Action Required:** Investigate exact tag structure in `.content` files.

### Critical Operations

| Operation | Implementation |
|-----------|----------------|
| **Move document** | Update `parent` field in `.metadata` to destination folder UUID |
| **Create folder** | Generate new UUID, create `.metadata` with `type: "CollectionType"` |
| **Rename** | Update `visibleName` in `.metadata` |
| **Delete** | Set `parent: "trash"` (soft delete) or `deleted: true` + create `.tombstone` |
| **Tag** | Update tags array in `.content` file |

---

## Proposed Implementation

### Phase 1: Core Write Operations

New tools to add:

#### `remarkable_move(document: str, destination: str) -> dict`
Move a document or folder to a new location.
- Find source UUID by name/path
- Find destination folder UUID
- Update `parent` field in source's `.metadata`
- Update `lastModified` timestamp
- Return confirmation with new path

#### `remarkable_create_folder(name: str, parent: str = "/") -> dict`
Create a new folder.
- Generate new UUID
- Create `.metadata` with `type: "CollectionType"`
- Create minimal `.content` file
- Return new folder details

#### `remarkable_rename(item: str, new_name: str) -> dict`
Rename a document or folder.
- Find item UUID
- Update `visibleName` in `.metadata`
- Return confirmation

#### `remarkable_delete(item: str, permanent: bool = False) -> dict`
Delete a document or folder.
- If `permanent=False`: Set `parent: "trash"`
- If `permanent=True`: Set `deleted: true`, create `.tombstone`
- Return confirmation

### Phase 2: Tag Management

#### `remarkable_tag(item: str, tags: list[str]) -> dict`
Add tags to a document or folder.
- Parse `.content` file
- Add tags to tags array
- Write back
- Return updated tags

#### `remarkable_untag(item: str, tags: list[str]) -> dict`
Remove tags from an item.

#### `remarkable_list_tags() -> dict`
List all unique tags in the library.

#### `remarkable_find_by_tag(tag: str) -> dict`
Find all items with a specific tag.

### Phase 3: Bulk Operations & Helpers

#### `remarkable_bulk_move(items: list[str], destination: str) -> dict`
Move multiple items at once.

#### `remarkable_organise_suggestions(folder: str = "/") -> dict`
Analyse a folder and suggest organisation improvements:
- Group similar documents
- Identify orphaned items
- Suggest folder structure
- Flag potential duplicates

### Phase 4: Interactive Organisation Mode

Build a workflow for collaborative organisation:
1. AI scans library structure
2. Presents current state and issues
3. Proposes organisation scheme
4. User approves/modifies
5. AI executes moves with confirmation

---

## Suggested Folder Structure for User

Based on user's context (Aviation Services Director, MBA student, technical projects):

### Top-Level Folders
```
/Work
  /Modini
    /Projects
    /Meetings
    /Reference
  /Policy & Regulation
  /Presentations
  
/MBA
  /Modules
  /Assignments
  /Reading

/Personal
  /Technical Projects
    /3D Printing
    /CAD
  /Planning
  /Reading

/Archive
  /2024
  /2023

/Inbox  (triage folder for new items)
```

### Suggested Tags
- `#active` - Currently working on
- `#reference` - Keep for reference
- `#review` - Needs review/action
- `#urgent` - Time-sensitive
- `#modini` - Work-related
- `#mba` - MBA-related
- `#technical` - Technical documentation
- `#meeting-notes` - Meeting notes
- `#annotated` - PDFs with annotations

---

## Technical Considerations

### Sync Behaviour
- After modifying files via SSH, the tablet may need to resync
- Changes should trigger `xochitl` to refresh (may need `systemctl restart xochitl`)
- Cloud sync will propagate changes to other devices

### Safety Measures
1. **Backup before bulk operations** - Export current state
2. **Dry-run mode** - Show what would change without executing
3. **Undo capability** - Track operations for potential rollback
4. **Confirmation prompts** - Require user approval for destructive operations

### Paper Pro Move Considerations
- Different CPU architecture (vs rM2) - but MCP runs on host, not device
- Check SSH/developer mode availability on Paper Pro Move
- Test file format compatibility

---

## Development Approach

### Getting Started
1. Fork `SamMorrowDrums/remarkable-mcp`
2. Set up development environment with `uv`
3. Connect to device via SSH for testing
4. Investigate `.content` file structure for tags

### Testing Strategy
1. Create test documents/folders on device
2. Manually verify filesystem changes
3. Build unit tests for metadata manipulation
4. Integration tests against actual device

### Contribution Strategy
- Develop in fork first
- Once stable, consider PR to upstream
- Issue #24 indicates maintainer interest in write support
- Follow existing code style and FastMCP patterns

---

## Open Questions

1. **Tag Structure:** Exact JSON format for tags in `.content` files?
2. **Sync Trigger:** Best way to trigger UI refresh after filesystem changes?
3. **Paper Pro Move:** Any differences in filesystem structure?
4. **Cloud Sync:** How do local changes propagate when cloud sync is enabled?
5. **Permissions:** Any files that need special handling on newer firmware?

---

## Resources

### Documentation
- [remarkable-mcp GitHub](https://github.com/SamMorrowDrums/remarkable-mcp)
- [Author's Blog Post](https://sam-morrow.com/blog/building-an-mcp-server-for-remarkable)
- [reMarkable Filesystem Info](https://remarkable.jms1.info/info/filesystem.html)
- [awesome-reMarkable](https://github.com/reHackable/awesome-reMarkable)
- [reMarkable Developer Docs](https://developer.remarkable.com/documentation)

### Related Projects
- [remarkable-fs](https://github.com/nick8325/remarkable-fs) - FUSE filesystem with write support
- [RMfuse](https://github.com/rschroll/rmfuse) - FUSE for reMarkable Cloud
- [rmapi](https://github.com/ddvk/rmapi) - reMarkable Cloud API client

### Libraries Used
- [rmscene](https://github.com/ricklupton/rmscene) - Parse .rm files
- [FastMCP](https://github.com/jlowin/fastmcp) - MCP server framework
- [PyMuPDF](https://pymupdf.readthedocs.io/) - PDF handling

---

## Next Steps

1. [ ] Fork the repository
2. [ ] Set up local development environment
3. [ ] Connect to reMarkable via SSH
4. [ ] Investigate `.content` file structure for native tags
5. [ ] Implement `remarkable_move()` as proof of concept
6. [ ] Test on both reMarkable 2 and Paper Pro Move
7. [ ] Iterate on remaining write operations
8. [ ] Build interactive organisation workflow

---

*Document created: February 2026*
*For use with Claude Code development*
