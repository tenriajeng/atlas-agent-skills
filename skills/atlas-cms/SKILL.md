---
name: atlas-cms
description: Use when integrating with or operating Atlas CMS - fetching entries, pages, blocks or media into a frontend, generating TypeScript types from a workspace schema, writing/publishing/scheduling content programmatically, uploading media, or operating content through the Atlas MCP tools (list_entries, create_entry, publish_page, etc.). Trigger on mentions of Atlas CMS, @latellu/atlas-sdk, @latellu/atlas-cli, atlas_live_/atlas_mgmt_ API keys, or api.atlas.latellu.com.
---

# Atlas CMS

A router, not a knowledge dump. Read the shared concepts below (always), then load
**only** the reference file for the surface you are actually using.

## Shared concepts

**Content model.** A *workspace* contains *content types* (schemas). A content type has
*entries* (structured records with a `slug`, a `status`, and a `data` field map). A
workspace also has *pages*: a page has SEO metadata and an ordered tree of *blocks*
(`{ id, type, position, data }`). Localized content is delivered as base-locale data plus
a `translations` array; readers merge a requested locale field-by-field with fallback to
the base locale.

**Two API key classes**, both sent as the `X-API-Key` header (never `Authorization`):

| Key prefix | Plane | Base path | Can do |
|---|---|---|---|
| `atlas_live_*` | Delivery (read) | `/api/v1/public` | Read published entries, pages, media, schema |
| `atlas_mgmt_*` | Management (write) | `/api/v1/manage` | Create/update/publish/delete content, upload media |

- Keys are minted in the dashboard: `cms.atlas.latellu.com/dashboard/api-keys`.
- Default base URL: `https://api.atlas.latellu.com` (override only for self-hosted).
- **Workspace scoping is implicit in the key.** There is no workspace parameter anywhere;
  a key reaches exactly one workspace.
- Management keys act as a service actor bound to the RBAC permissions of the person who
  created them, further limited by scopes: `content:write`, `content:publish`,
  `media:write`. A missing scope returns 403 on that specific operation.
- Production `atlas_live_*` keys only ever see **published** content. Draft content is
  not reachable through them, regardless of query parameters.

**Safety rule:** never ship an `atlas_mgmt_*` key or the management client in
browser-delivered code. Management usage is server-side only (API routes, scripts, CI).

## Which surface do I use?

| You are... | Use | Read |
|---|---|---|
| Building a frontend that renders Atlas content | SDK delivery client | [references/sdk-delivery.md](references/sdk-delivery.md) |
| Wanting typed content interfaces for the SDK | CLI type generator | [references/cli-typegen.md](references/cli-typegen.md) |
| Writing content from your own code (scripts, migrations, backends) | SDK management client | [references/sdk-management.md](references/sdk-management.md) |
| Operating content via MCP tools in this session (`list_entries`, `create_entry`, ...) | Atlas MCP server | [references/mcp.md](references/mcp.md) |

Do **not** read references you don't need. If a task spans surfaces (e.g. "generate
types, then build the page"), route each sub-task to its own single reference.

## Extending

Deep API detail lives at https://docs.atlas.latellu.com — link there rather than
duplicating endpoint documentation into this skill.
