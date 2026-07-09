# Atlas MCP tools

The `@latellu/atlas-mcp` server (stdio) exposes Atlas content operations as agent tools.
This file describes `@latellu/atlas-mcp` **≥ 1.2.0** (the bundled `.mcp.json` uses
`npx -y`, so it always runs the latest). Older servers lack `translations`,
`seo_translations`, per-field error detail, the schema `id`/`is_block` fields, page/entry lifecycle tools (unpublish/archive/duplicate/schedule), bulk operations, reorder tools, media alt-text editing, list meta+cursor pagination, automatic 429 retry, and pre-parsed block data in get_page.

**Environment** — read tools need `ATLAS_LIVE_API_KEY`, write tools need
`ATLAS_MGMT_API_KEY`; with only one set, the other half of the tools fails client-side
naming the missing variable, and with neither set the server exits at startup.
`ATLAS_API_URL` overrides the base URL (self-hosted). `MCP_ALLOWED_UPLOAD_PATHS`
(comma-separated absolute dirs) restricts where `upload_media` may read files from.

Tool results are the API's `data` payload as pretty-printed JSON text; failures come
back as `Error: Atlas API error: <status> - <message>` tool results (not protocol
errors).

## Tool catalog

### Schema & discovery — live key

| Tool | Params | Returns |
|---|---|---|
| `get_workspace_schema` | — | `{ workspace: { slug, name, locales, default_locale }, content_types: [...] }` — each content type carries `id` and `is_block`; types with `is_block: true` are the valid `block_type_id` values for page blocks |
| `list_content_types` | — | `content_types` array (client-side view of the schema) |
| `get_content_type` | `content_type` | one content type with its full field list; error text if the slug doesn't exist |

### Entries — read: live key · write: mgmt key

| Tool | Params | Notes |
|---|---|---|
| `list_entries` | `content_type`, `page?`=1, `limit?`=20, `status?`, `cursor?` | `status` is enum (draft \| published \| archived \| scheduled); live key **cannot** reveal drafts. Returns `{ data, meta }` where `meta` has `total_data`, `total_pages` (offset mode) or `next_cursor` (cursor mode). Use `cursor: meta.next_cursor` to page past the 1000-page limit. |
| `get_entry` | `content_type`, `slug` | resolves by `slug` alone — `content_type` is accepted but errors clearly if the slug belongs to a different content type |
| `create_entry` | `content_type`, `data`, `slug?`, `translations?` | creates a **draft**; slug auto-generated when omitted; `translations` = `{locale: {data: {...}}}` (required+localizable fields go HERE) |
| `update_entry` | `content_type`, `slug`, `data`, `translations?` | **full replace** of `data` (and per-locale `translations`), not a patch |
| `publish_entry` / `unpublish_entry` / `archive_entry` | `content_type`, `slug` | need the `content:publish` scope |
| `duplicate_entry` | `content_type`, `slug` | copies to a new draft |
| `schedule_entry` | `content_type`, `slug`, `publish_at` | schedules publish for a future time (ISO 8601 UTC); needs `content:publish` scope |
| `bulk_entries` | `content_type`, `operations` | batch publish/archive/schedule/delete by entry UUIDs; each op in array: `{ op, id, publish_at? }` |
| `bulk_delete_entries` | `content_type`, `ids` | batch delete by entry UUIDs |
| `reorder_entries` | `content_type`, `ids` | reorder entries; pass the complete ordered list of entry UUIDs |
| `delete_entry` | `content_type`, `slug` | returns JSON `{ deleted: true, ... }` |

### Pages — read: live key · write: mgmt key

| Tool | Params | Notes |
|---|---|---|
| `list_pages` | `page?`=1, `limit?`=20, `cursor?` | summaries, no blocks. Returns `{ data, meta }` with cursor pagination option (same as `list_entries`). |
| `get_page` | `slug` | block tree — each block's `data` is already parsed to an object (no JSON.parse needed); nested children and translations included |
| `create_page` | `slug`, `title?`, `blocks?`, `seo?`, `seo_translations?` | draft; `title` is an alias for `seo.title`; `seo` accepts `title`/`description`/`keywords`/`og_image`/`canonical`; blocks = `[{block_type_id, parent_id?, position, data, translations?}]` |
| `update_page` | `slug`, `title?`, `blocks?`, `seo?`, `seo_translations?` | passing `blocks` replaces the whole list; `seo` replaces wholesale (title alone drops other seo fields) |
| `publish_page` | `slug` | publish a page; fails on blockless pages |
| `unpublish_page` | `slug` | unpublish a page; needs `content:publish` scope |
| `archive_page` | `slug` | archive a page; needs `content:publish` scope |
| `duplicate_page` | `slug` | copy to a new draft |
| `schedule_page` | `slug`, `publish_at` | schedule publish for a future time (ISO 8601 UTC); needs `content:publish` scope |
| `reorder_page_blocks` | `slug`, `block_ids` | reorder blocks; pass the complete ordered list of block IDs |
| `delete_page` | `slug` | returns JSON `{ deleted: true, ... }` |

### Media

| Tool | Params | Notes |
|---|---|---|
| `get_media` | `id` | live key |
| `upload_media` | `file_path` (absolute), `folder?`="/", `alt_text?` | mgmt key; reads the local file, multipart upload. Pre-validates client-side: ≤10 MB, extension allowlist (.jpg .jpeg .png .gif .webp .mp4 .webm .mov .pdf). |
| `update_media` | `id`, `alt_text?`, `folder?` | mgmt key; edit metadata (alt text and/or folder). |
| `delete_media` | `id` | mgmt key; returns JSON `{ deleted: true, ... }` |

## Recommended write workflow

1. **Schema first.** Call `get_content_type` (or `get_workspace_schema`) before any
   `create_entry`/`update_entry`. The `data` argument is an unvalidated map; the backend
   rejects wrong shapes with 400. Error results include per-field detail
   (`title: title is required`) and a `traceId` — read them instead of guessing.
   Knowing the exact field names/types up front is still the difference between one
   call and five.
2. **Read-modify-write for updates.** `update_entry` replaces `data` wholesale:
   `get_entry` → merge your change into the full object → `update_entry` with everything.
3. **Create → verify → publish.** `create_entry` makes a draft; `publish_entry` needs
   the `content:publish` scope (403 without it, with a clear message).
4. **Before `upload_media`**: the tool pre-validates client-side (≤ 10 MB, extension
   allowlist), so failures are caught before upload. Allowed extensions:
   `.jpg .jpeg .png .gif .webp .mp4 .webm .mov .pdf`.

## Capability gaps — route elsewhere instead of retrying

The MCP manage plane is now fully covered. These gaps remain:

- **No automatic merge on `update_entry`** — the tool replaces `data` wholesale, so you
  must read-modify-write yourself: `get_entry` → merge your changes → `update_entry`.
  (The management SDK's `update` is a full replace too — this is API semantics, not an
  MCP quirk.)
- **Schema is dashboard-only.** Creating content types, fields, or block types is not
  exposed via MCP, SDK, or API. "Create a content type" = ask the user to do it in the
  dashboard, then continue.

Before composing any `data` payload, read `authoring.md` — it defines the exact value
format per field type (richtext = Tiptap HTML, relation/image = UUIDs, etc.).

## Gotchas

- **Drafts are invisible to live keys** — `list_entries(status="draft")` returning
  empty means key scoping, not an empty workspace.
- **`get_entry` looks up by slug alone** (slug-global, `content_type` is not sent to the
  API) — but if the entry found belongs to a different content type than you requested,
  the tool returns a clear mismatch error instead of the wrong entry.
- Every write sends a fresh auto-generated `Idempotency-Key` per call, so re-issuing a
  failed create as a *new tool call* is a new operation — check whether the first call
  actually landed (e.g. `get_entry`) before retrying creates.
- One MCP server = one workspace (the key's). Multi-workspace work needs one server
  registration per workspace.
- **The server never checks that the live and mgmt keys belong to the same workspace.**
  A mismatched pair is maximally confusing: schema and reads come from workspace A while
  every write 404s ("content type not found") against workspace B. If reads succeed but
  writes 404 on types you just listed, suspect mismatched keys before anything else.
