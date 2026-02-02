# reMarkable Filesystem Exploration Notes

**Device:** Paper Pro Move (imx93-chiappa)
**Firmware:** 20260129075305 (January 29, 2026)
**Total Items:** 122

## Directory Structure

Base path: `/home/root/.local/share/remarkable/xochitl/`

## File Types Per Item

Each item (document or folder) has a UUID and associated files:

```
{UUID}.metadata     - Core metadata (name, type, parent, timestamps)
{UUID}.content      - Page info, tool settings, tags
{UUID}.template     - Template info (for template items)
{UUID}.pagedata     - Page template assignments
{UUID}.thumbnails/  - Thumbnail images directory
{UUID}/             - Directory with .rm files (stroke data per page)
{UUID}.pdf          - Original PDF (if uploaded)
{UUID}.epub         - Original EPUB (if uploaded)
```

## Metadata File Structure

### CollectionType (Folder)
```json
{
    "createdTime": "1731321686123",
    "lastModified": "1731321686123",
    "parent": "d180555f-1111-4e65-8efb-9e3c2ea6ddc1",
    "pinned": false,
    "type": "CollectionType",
    "visibleName": "Internal T&E"
}
```

### DocumentType
```json
{
    "createdTime": "1752739810657",
    "lastModified": "1752828064038",
    "lastOpened": "1752764452386",
    "lastOpenedPage": 1,
    "new": false,
    "parent": "d180555f-1111-4e65-8efb-9e3c2ea6ddc1",
    "pinned": false,
    "source": "",
    "type": "DocumentType",
    "visibleName": "CFASC"
}
```

### TemplateType
```json
{
    "createdTime": "1740131363000",
    "lastModified": "1740131363000",
    "new": false,
    "parent": "",
    "pinned": false,
    "source": "com.remarkable.methods",
    "type": "TemplateType",
    "visibleName": "Task priority"
}
```

## Content File Structure

### CollectionType .content (Folder)
```json
{
    "tags": []
}
```

Very simple - just contains a tags array.

### DocumentType .content (Notebook/PDF)
Much more complex, includes:
- `cPages` - Page information with IDs, templates, scroll positions
- `extraMetadata` - Tool settings (LastPen, colors, sizes, etc.)
- `documentMetadata` - Document-specific metadata
- `fileType` - "notebook", "pdf", etc.
- `formatVersion` - Currently seeing version 2
- `pageCount` - Number of pages
- `tags` - Array of tags (currently empty in all examined documents)
- `pageTags` - Array of page-specific tags (also empty)
- Various display settings (orientation, zoom, etc.)

## Key Findings

### Tags - CRITICAL DISCOVERY

reMarkable supports **two types of tags**:

**UI Workflows:**
- **Page Tag**: Created when tagging FROM WITHIN a document (tags current page)
- **Document Tag**: Created when long-pressing a document FROM THE LIBRARY LIST

#### 1. Page Tags (ONLY type used by UI)
- **Location:** `pageTags` array in `.content` files
- **Purpose:** Tag specific pages within a document
- **UI Behavior:** When you "tag a document" in the reMarkable UI, it tags **the currently displayed page** (whatever page you're viewing when creating the tag)
- **Format:**
  ```json
  "pageTags": [
      {
          "name": "Follow up",
          "pageId": "0b6e53b7-3cbf-49bf-859a-5293b1cf3566",
          "timestamp": 1732190732505
      },
      {
          "name": "DART",
          "pageId": "cbb40629-65b1-47ef-bd38-f5a4fbb233c9",
          "timestamp": 1770032825572
      }
  ]
  ```
- **Fields:**
  - `name` (string): Tag name as displayed in UI
  - `pageId` (string): UUID of the page being tagged (corresponds to page ID in `cPages` array)
  - `timestamp` (number): Unix timestamp in milliseconds when tag was created/modified
- **Important:** The pageId matches the `id` field in the `cPages.pages` array, which can be mapped to page numbers via the `redir` value or position in pages array

#### 2. Document Tags (Used when long-pressing documents in library)
- **Location:** `tags` array in `.content` files
- **UI Workflow:** Long-press a document in the library/document list view
- **Format:**
  ```json
  "tags": [
      {
          "name": "doxrwF",
          "timestamp": 1770033656154
      }
  ]
  ```
- **Fields:**
  - `name` (string): Tag name as displayed in UI
  - `timestamp` (number): Unix timestamp in milliseconds when tag was created/modified
  - **Note:** NO `pageId` field - tag applies to entire document
- **Difference from Page Tags:** Simpler structure without page reference

#### Examples Found

**Document:** "DART250-A-010QTP_Mission Avionics (REV-08)" (d14cebdf-e4d8-4947-847f-c2fe4efc33b9)
- Has **multiple page tags** on page ID cbb40629-65b1-47ef-bd38-f5a4fbb233c9:
  - "DART" (timestamp: 1770032825572)
  - "Document" (timestamp: 1770032980905)
- Demonstrates: Multiple tags can be applied to the same page

**Document:** "Next Steps" (af845f66-de10-4bd3-9d10-f6d2262f9257)
- Has page tag "Follow up" on page ID 0b6e53b7-3cbf-49bf-859a-5293b1cf3566

### Folder Hierarchy
Sample folder structure observed:
- Top-level folders: `"5 People"`, `"Pers"`, `"4 Archive"`, `"1 Projects"`, `"2 Areas"`
- Nested folders under various parents
- PARA-style organization with numbered prefixes
- Parent field uses UUID or `""` for root level

### Parent Field Values
- `""` (empty string) - Root level
- `{UUID}` - Inside a specific folder
- `"trash"` - In trash (for soft delete)

## Questions to Answer

1. ~~**Tag Format:**~~ ✅ **ANSWERED**
   - Page tags use object format: `{"name": "tag_name", "pageId": "page_uuid", "timestamp": milliseconds}`
   - Document tags format still unknown (array appears unused by UI)

2. ~~**Page Tags:**~~ ✅ **ANSWERED**
   - Used for tagging specific pages within documents
   - This is what the UI currently creates when you "tag" something

3. ~~**Document Tags:**~~ ✅ **PARTIALLY ANSWERED**
   - UI never uses the `tags` array - only `pageTags`
   - Even "document tags" created in UI go to `pageTags`
   - Structure likely reserved for:
     - Future firmware feature
     - API/programmatic use (which we could leverage!)
   - **Decision needed:** Should our MCP implement document-level tags for simpler AI organization?

4. ~~**Sync Behavior:**~~ ✅ **SOLVED**
   - **xochitl maintains internal state** that gets written when documents are accessed
   - Initial attempt failed because user opened document while xochitl was running
   - Opening document triggered xochitl to regenerate .content from internal state
   - **SOLUTION FOUND:** Stop xochitl before modifying files!

   **Successful Workflow:**
   ```bash
   systemctl stop xochitl
   # Modify .content and .metadata files
   systemctl start xochitl
   ```

   **Verified:** Tag appears correctly in UI after following this workflow! ✅

5. **Cloud Sync:** How do local changes propagate to reMarkable Cloud?
   - Still untested - need to understand validation first
   - xochitl may sync with cloud and revert local-only changes

## User's Current Organization

The user has implemented a PARA-style system:
- **1 Projects** - Active project work
- **2 Areas** - Ongoing responsibilities
- **4 Archive** - Completed items
- **5 People** - People-related notes
- **Pers** - Personal items

With project-specific folders like:
- Anzen Flying, SNIPE, SULTAN, Geodrones, BRAKESTOP, DART 250, HERA, Saudi
- Internal T&E, Consultancy

## Implementation Implications for MCP

### Tag Management Functions

Based on findings, we should implement:

#### Page Tag Functions (Primary Implementation)

1. `remarkable_tag_page(document: str, page: int, tag: str) -> dict`
   - Add a tag to a specific page
   - Look up page UUID from `cPages.pages` array using page number
   - Add entry to `pageTags` array with current timestamp
   - Return confirmation with tag details

2. `remarkable_tag(document: str, tag: str, page: int = 0) -> dict`
   - Convenience wrapper - defaults to page 0 for "document-level" tagging
   - Mirrors UI behavior where you tag "the current page"
   - Most users will likely tag page 0 to represent the whole document

3. `remarkable_untag_page(document: str, page: int, tag: str) -> dict`
   - Remove specific tag from a page
   - Filter `pageTags` array by matching name and pageId

4. `remarkable_list_tags(document: str, page: int | None = None) -> dict`
   - If page specified: return tags for that page
   - If page=None: return all tags across all pages (grouped by page)

5. `remarkable_find_by_tag(tag: str) -> dict`
   - Search all documents for pages with specific tag
   - Return list of (document, page_number, page_id) tuples

6. `remarkable_list_all_tags() -> dict`
   - Return unique list of all tag names across entire library
   - Useful for autocomplete/suggestions

### File Modification Pattern ✅ PROVEN

**Successful workflow for adding a tag:**

1. **Stop xochitl:** `systemctl stop xochitl`
2. **Read `.content` file** and parse JSON
3. **Add entry to `pageTags` array:**
   ```json
   {
       "name": "tag_name",
       "pageId": "page_uuid_from_cPages",
       "timestamp": current_time_in_milliseconds
   }
   ```
4. **Update `.metadata`:**
   - Set `lastModified` to current timestamp (as string)
5. **Write both files back**
6. **Start xochitl:** `systemctl start xochitl`
7. **Result:** Tag appears in UI! ✅

**For document tags:** Same process but modify `tags` array instead of `pageTags` (no `pageId` field needed)

### Safety Considerations
- Always backup `.content` before modification
- Validate JSON structure before writing
- Handle concurrent modification (file locking?)
- Test sync behavior thoroughly

## Next Steps

1. ~~Create test tags via reMarkable UI~~ ✅ Done
2. ~~Examine populated tag structure~~ ✅ Done
3. **Test programmatically adding a page tag**
   - Add a new tag to an existing document
   - Verify it appears in UI
   - Check if restart needed
4. **Test programmatically adding a document tag**
   - Add string to `tags` array
   - See if UI recognizes it
5. Implement write operations based on findings
6. Test on both reMarkable 2 and Paper Pro Move (currently testing on Paper Pro Move)
