---
name: atlas-cms
description: Use for any task that touches Atlas CMS, however casually phrased - fetching entries, pages, blocks or media into a frontend, generating/gen-ing TypeScript types from a workspace schema, writing/publishing/scheduling/pushing content programmatically, uploading media, or operating content through the Atlas MCP tools (list_entries, create_entry, publish_page, etc.). Always trigger on any mention of Atlas CMS, @latellu/atlas-sdk, @latellu/atlas-cli, atlas_live_/atlas_mgmt_ API keys, or api.atlas.latellu.com - including quick/informal asks like "gen the types for our repo" or "push some content into atlas real quick" - rather than answering from general knowledge, which risks hallucinating wrong package names or endpoints.
---

# Atlas CMS

A router, not a knowledge dump. Read the shared concepts below (always), then load
**only** the reference file for the surface you are actually using.

## Content model

A **workspace** is the tenant — it owns everything below and is reached by exactly one
API key (there is no workspace parameter on any endpoint or tool; scoping is implicit in
the key).

- **Content type** — a schema (e.g. "Article") defining fields. Field types: `text`,
  `textarea`, `richtext`, `number`, `boolean`, `date`, `select` (with options),
  `image`, `relation`, `content_type_reference`.
- **Entry** — a record of a content type: `{ id, slug, status, published_at, data }`
  where `data` is the field map. Lifecycle: `draft → published → archived`, plus
  `scheduled` (publish at a future time).
- **Page** — like an entry but composed of ordered **blocks**
  (`{ id, type, position, data }`) and carrying its own SEO object
  (`title, description, keywords, og_image, canonical`).
- **Locale** — content is stored as base-locale data plus a `translations` array.
  Readers merge the requested locale **field-by-field**; untranslated fields fall back
  to the base locale. `image`/`relation` values are IDs/slugs, never resolved objects.

## Authentication

Two key classes, both sent as the `X-API-Key` header (never `Authorization: Bearer`):

| Key prefix | Plane | Base path | Can do |
|---|---|---|---|
| `atlas_live_*` | Delivery (read) | `/api/v1/public` | Read **published** entries, pages, media, schema |
| `atlas_mgmt_*` | Management (write) | `/api/v1/manage` | Create/update/publish/delete content, upload media |

- Keys are minted at `cms.atlas.latellu.com/dashboard/api-keys` (Developer → API Keys).
- Default base URL: `https://api.atlas.latellu.com` (override only for self-hosted).
- The planes are strict: a live key gets 401 on `/manage/*`, a mgmt key gets 401 on
  `/public/*`. A live key sees **only published content** — no query parameter reveals
  drafts.
- Management keys act as a service actor bound to the RBAC permissions of whoever
  created them, further limited by scopes: `content:write` (create/update/delete),
  `content:publish` (publish/unpublish/archive/schedule), `media:write` (uploads).
  A missing scope → 403 on that operation only.

**Safety rule:** never ship an `atlas_mgmt_*` key or the management client in
browser-delivered code. Management usage is server-side only (API routes, scripts, CI).

## Wire format (applies to every surface)

Every REST response is an envelope: `{ success, message, data?, meta?, code?, errors? }`.
The SDK and MCP unwrap it, but two payload quirks leak through in places: `entry.data`,
page `seo`, and block `data` are stored/transported as **JSON strings** (the SDK and
MCP ≥ 1.2.0 parse them for you), and list `meta` uses
`{ total_data, current_page, page_size, total_pages, next_cursor }`.

Error codes you will actually see: `UNAUTHORIZED` (401 — missing/empty/wrong-class key),
`FORBIDDEN` (403 — missing scope or RBAC), `NOT_FOUND` (404 — also returned for
other-workspace resources), `VALIDATION_ERROR` (400 — with field detail),
`PAGE_TOO_DEEP` (400 — offset pagination past page 1000), and 429 rate limiting
(the SDK management client and MCP ≥ 1.2.0 retry automatically; elsewhere back off
exponentially yourself).

Pagination limits: `limit` max 100, `page` max 1000; past that, switch to cursor
pagination (`meta.next_cursor`, opaque — never construct cursors yourself; `total_data`
is `-1` in cursor mode).

## Which surface do I use?

| You are... | Use | Read |
|---|---|---|
| Building a frontend that renders Atlas content | SDK delivery client | [references/sdk-delivery.md](references/sdk-delivery.md) |
| Wanting typed content interfaces for the SDK | CLI type generator | [references/cli-typegen.md](references/cli-typegen.md) |
| Writing content from your own code (scripts, migrations, backends) | SDK management client | [references/sdk-management.md](references/sdk-management.md) |
| Operating content via MCP tools in this session (`list_entries`, `create_entry`, ...) | Atlas MCP server | [references/mcp.md](references/mcp.md) |
| Composing any `data` payload — field value formats, translations, page blocks | Authoring guide | [references/authoring.md](references/authoring.md) |

Do **not** read references you don't need. If a task spans surfaces (e.g. "generate
types, then build the page"), route each sub-task to its own single reference.

Choosing between MCP and the management SDK for writes: MCP is for operating content
interactively in an agent session (one-off edits, content entry, audits); the SDK is for
code that outlives the session (migrations, sync jobs, backends) and for structured per-field error objects.
This is about scale and recoverability, not run count: if the task touches more than a
handful of entries, needs to survive partial failure and be safely re-run, or needs
structured per-field error handling, that's the SDK management client — even if you'll
only run it once. A 200-entry backfill script is an SDK job; MCP's "one-off" framing
means a few manual edits typed out in conversation, not a bulk script.

## Hard capability boundaries (don't go looking for these)

- **Schema is dashboard-only.** No API, SDK, CLI, or MCP surface creates content types,
  fields, or block types. "Create a content type" = ask the user to do it in the
  dashboard, then continue.
- **No field-value queries.** The public list endpoints filter by `type`/`locale` only —
  "all articles where category = X" means paging through and filtering client-side.
- **The management plane is write-only** (no GET endpoints). Reading always needs a
  live key; a write-then-verify flow therefore needs both keys.

Deep API detail lives at https://docs.atlas.latellu.com — link there rather than
duplicating endpoint documentation into this skill.
