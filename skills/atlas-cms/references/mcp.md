# Atlas MCP tools

The plugin bundles the `@latellu/atlas-mcp` server (stdio). Read tools need
`ATLAS_LIVE_API_KEY`; write tools need `ATLAS_MGMT_API_KEY`. With only one key set, the
other half of the tools fails client-side naming the missing variable. If neither is set
the server exits at startup. `ATLAS_API_URL` overrides the base URL for self-hosted.

## Tool catalog

**Schema & discovery** (live key)
- `get_workspace_schema` — workspace meta (slug, locales, default locale) + all content types.
- `list_content_types` / `get_content_type(content_type)` — convenience views over the schema.

**Entries** (read: live key; write: mgmt key)
- `list_entries(content_type, page?, limit?, status?)`
- `get_entry(content_type, slug)`
- `create_entry(content_type, data, slug?)` — creates a **draft**; slug auto-generated if omitted.
- `update_entry(content_type, slug, data)`
- `publish_entry` / `unpublish_entry` / `archive_entry` / `duplicate_entry` / `delete_entry` — all `(content_type, slug)`.

**Pages** (read: live; write: mgmt)
- `list_pages(page?, limit?)` / `get_page(slug)`
- `create_page(title, slug?, blocks?, seo?)` — creates a draft. `seo` accepts `title`/`description` only.
- `update_page(slug, title?, blocks?, seo?)` / `publish_page(slug)` / `delete_page(slug)`

**Media**
- `get_media(id)` (live key)
- `upload_media(file_path, folder?, alt_text?)` (mgmt key) — reads a local file and uploads it.

## Recommended workflow for writes

1. Call `get_content_type` (or `get_workspace_schema`) first to learn the exact field
   names and types. `data` is an unvalidated map — the backend rejects bad shapes with a
   400, and per-field validation detail is **not** surfaced through MCP (you only see
   `Atlas API error: 400 - validation failed`).
2. `create_entry` / `update_entry` with a `data` object matching the schema.
3. `publish_entry` when ready (requires the `content:publish` scope on the key).

## Gotchas

- **`update_entry` is a full replace** of `data`, not a patch. Read the entry first,
  merge your change, then send the complete object (read-modify-write).
- **Page lifecycle is asymmetric**: there is no `unpublish_page`, `archive_page`, or
  `duplicate_page` tool. There are also no bulk, schedule, or block-reorder tools, and no
  `delete_media`/`update_media`. For those, use the dashboard or the SDK management
  client (see `sdk-management.md`).
- **`get_page` returns each block's `data` as a JSON-encoded string**, not an object.
  Parse it before reading fields — otherwise blocks look empty.
- **`get_entry` resolves by slug alone**; its `content_type` argument is accepted but not
  sent to the backend. Don't rely on it to disambiguate slugs shared across types.
- **Drafts are invisible to live keys.** `list_entries(status="draft")` with a production
  key returns nothing — that is key scoping, not an empty workspace.
- **Uploads are validated server-side only**: max 10 MB; MIME must be `image/*`,
  `video/*`, or `application/pdf`. MIME is inferred from the file extension; unknown
  extensions become `application/octet-stream` and are always rejected (e.g. `.svg`,
  `.heic`). Check size and extension before calling `upload_media`. If
  `MCP_ALLOWED_UPLOAD_PATHS` is set, `file_path` must live under one of those directories.
- **No client-side retry on 429.** On `Atlas API error: 429 - ...`, back off
  (exponentially) and retry yourself.
- **`delete_entry` / `delete_page` return plain text** ("Entry 'x' deleted successfully"),
  unlike every other tool, which returns JSON.
- Every write automatically sends a fresh `Idempotency-Key` header per call; identical
  retries you issue as *new tool calls* are new operations, not deduplicated.
