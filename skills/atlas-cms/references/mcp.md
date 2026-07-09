# Atlas MCP tools

The `@latellu/atlas-mcp` server (stdio) exposes Atlas content operations as agent tools.
This file describes `@latellu/atlas-mcp` **≥ 1.1.0** (the bundled `.mcp.json` uses
`npx -y`, so it always runs the latest). Older servers lack `translations`,
`seo_translations`, per-field error detail, and the schema `id`/`is_block` fields.

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
| `list_entries` | `content_type`, `page?`=1, `limit?`=20, `status?` | `status` is free text and **cannot** reveal drafts to a live key |
| `get_entry` | `content_type`, `slug` | resolves by `slug` alone — `content_type` is accepted but not sent |
| `create_entry` | `content_type`, `data`, `slug?`, `translations?` | creates a **draft**; slug auto-generated when omitted; `translations` = `{locale: {data: {...}}}` (required+localizable fields go HERE) |
| `update_entry` | `content_type`, `slug`, `data`, `translations?` | **full replace** of `data` (and per-locale `translations`), not a patch |
| `publish_entry` / `unpublish_entry` / `archive_entry` | `content_type`, `slug` | need the `content:publish` scope |
| `duplicate_entry` | `content_type`, `slug` | copies to a new draft |
| `delete_entry` | `content_type`, `slug` | returns plain text, not JSON |

### Pages — read: live key · write: mgmt key

| Tool | Params | Notes |
|---|---|---|
| `list_pages` | `page?`=1, `limit?`=20 | summaries, no blocks |
| `get_page` | `slug` | block tree — each block's `data` is a **JSON string**, parse it |
| `create_page` | `slug`, `title?`, `blocks?`, `seo?`, `seo_translations?` | draft; `title` is an alias for `seo.title`; `seo` accepts `title`/`description`/`keywords`/`og_image`/`canonical`; blocks = `[{block_type_id, parent_id?, position, data, translations?}]` |
| `update_page` | `slug`, `title?`, `blocks?`, `seo?`, `seo_translations?` | passing `blocks` replaces the whole list; `seo` replaces wholesale (title alone drops other seo fields) |
| `publish_page` | `slug` | the ONLY page lifecycle tool — see gaps below; fails on blockless pages |
| `delete_page` | `slug` | plain-text result |

### Media

| Tool | Params | Notes |
|---|---|---|
| `get_media` | `id` | live key |
| `upload_media` | `file_path` (absolute), `folder?`="/", `alt_text?` | mgmt key; reads the local file, multipart upload |
| `delete_media` | `id` | mgmt key; plain-text result |

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
4. **Before `upload_media`**: check the file is ≤ 10 MB and has a known extension.
   Validation is entirely server-side; MIME is inferred from the extension and unknown
   extensions upload as `application/octet-stream`, which the backend always rejects
   (so `.svg`, `.heic` fail). Allowed: `image/*`, `video/*`, `application/pdf` (i.e.
   `.jpg .jpeg .png .gif .webp .mp4 .webm .mov .pdf`).

## Capability gaps — route elsewhere instead of retrying

These exist in the backend/SDK but have **no MCP tool**; say so and point to the
dashboard or the management SDK (`sdk-management.md`) rather than improvising:

- Page lifecycle beyond publish: **no `unpublish_page`, `archive_page`,
  `duplicate_page`**.
- **No** bulk operations, entry/page **scheduling**, or block **reorder** tools.
- **No `update_media`** (alt-text edits) — media is upload/read/delete through MCP.
- Pagination is offset-only (`page`/`limit`); there is no cursor option, so listings
  past page 1000 (`PAGE_TOO_DEEP`) are unreachable via MCP.

Before composing any `data` payload, read `authoring.md` — it defines the exact value
format per field type (richtext = Tiptap HTML, relation/image = UUIDs, etc.).

## Gotchas

- **Drafts are invisible to live keys** — `list_entries(status="draft")` returning
  empty means key scoping, not an empty workspace.
- **`get_page` block `data` needs `JSON.parse`** per block, or blocks look empty.
- **`get_entry` ignores `content_type`** — slugs shared across types resolve to
  whatever the backend finds by slug.
- **No client-side 429 retry** — on `Atlas API error: 429`, back off exponentially and
  retry the tool call yourself.
- Every write sends a fresh auto-generated `Idempotency-Key` per call, so re-issuing a
  failed create as a *new tool call* is a new operation — check whether the first call
  actually landed (e.g. `get_entry`) before retrying creates.
- One MCP server = one workspace (the key's). Multi-workspace work needs one server
  registration per workspace.
- **The server never checks that the live and mgmt keys belong to the same workspace.**
  A mismatched pair is maximally confusing: schema and reads come from workspace A while
  every write 404s ("content type not found") against workspace B. If reads succeed but
  writes 404 on types you just listed, suspect mismatched keys before anything else.
