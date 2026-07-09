# Authoring valid content (field formats, translations, page blocks)

What actually gets validated on write, what each field type's value must look like, and
how to write localized content. Applies to both the management SDK and the MCP write
tools — read this before composing any `data` payload.

## What the API validates (and what it silently accepts)

On entry create/update the backend enforces exactly four rules:

1. `required` — the field must be present and non-empty. For **richtext**, the Tiptap
   empty states `""`, `"<p></p>"`, `"<p><br></p>"` count as empty.
2. `unique` — value must not collide with another entry of the workspace.
3. `relation` fields — value must be a valid **entry UUID** (or array of UUIDs) that
   exists in the workspace. Slugs are rejected.
4. `image` fields — value must be a valid **media UUID** (or array) that exists in the
   workspace. Upload first (`media.upload` / `upload_media`), then reference the
   returned `id`.

**Everything else is stored as-is.** A number in a text field, a garbage date, a select
value outside the options — none of it is rejected. Validation failures you don't get
from the API become rendering bugs in the dashboard/frontend, so discipline comes from
you: fetch the schema first and match each field's type exactly.

## Value format per field type

| Field type | Write this | Notes |
|---|---|---|
| `text`, `textarea` | plain string | |
| `richtext` | **Tiptap HTML string**, e.g. `"<p>Hello <strong>world</strong></p>"` | not markdown, not JSON; the dashboard editor is Tiptap |
| `number` / `boolean` | JSON number / boolean | not validated — don't send strings |
| `date` | ISO 8601 string, `"2026-08-01"` (or full timestamp) | not format-validated; ISO keeps the dashboard picker working |
| `select` | one of the schema's `options`, exact string | not validated — off-list values break the generated literal-union types |
| `image` | media UUID string, or array of them | validated against workspace media |
| `relation`, `content_type_reference` | entry UUID string, or array | validated against workspace entries; **UUID, not slug** |

Reading back: the value is whatever was stored. Content created through this API holds
UUIDs for image/relation; older or imported datasets may hold URLs/slugs — when
rendering, branch on the shape (e.g. `value.startsWith("http")` → use directly,
otherwise resolve via `media.get(id)`).

## Writing translations (localized entries)

Entry create/update accept a `translations` map alongside `data`:

```jsonc
{
  "slug": "hello",
  "data":         { "title": "Hello",  "views": 10 },   // base locale
  "translations": {
    "de": { "data": { "title": "Hallo" } },              // only localizable fields
    "ja": { "data": { "title": "こんにちは" } }
  }
}
```

Rules that will bite you if unknown:

- **A `required` + `localizable` field is validated against `translations`, not
  `data`** — it must have a non-empty value in at least one translation locale, or the
  write fails with "required localizable field ... is missing".
- Translation `data` should contain **only localizable fields**; non-localizable values
  live in base `data` and fall back automatically on read.
- **Via the management SDK**: pass `translations` as an extra key on
  `CreateEntryInput`/`UpdateEntryInput` (the input types accept extra keys):

  ```ts
  await client.entries("article").update("hello", {
    data: fullBaseData,                                   // full replace — include everything
    translations: { de: { data: { title: "Hallo" } } },   // also a full replace per locale
  });
  ```

- **Via MCP: not possible.** The `create_entry`/`update_entry` tools forward only
  `slug` and `data` — a `translations` key is dropped. Translating content needs the
  SDK or the dashboard; say so instead of trying.

## Writing pages and blocks

The real write shape (`POST/PUT /pages`) differs from what the reading API shows:

```jsonc
{
  "slug": "landing",                       // required on create; there is NO "title" field —
  "seo": {                                 //   the page's title lives in seo.title
    "title": "Landing",
    "description": "...",
    "keywords": ["a", "b"],                // array of strings
    "og_image": "https://...",
    "canonical": "https://..."
  },
  "seo_translations": { "de": { "title": "Landung", ... } },
  "blocks": [
    {
      "block_type_id": "<UUID>",           // required per block — see warning below
      "parent_id": null,                    // block UUID for nesting, or null
      "position": 0,
      "data": { "...": "block fields" },
      "translations": { "de": { "data": { "...": "..." } } }
    }
  ]
}
```

- `update` with `blocks` replaces the block list; reordering alone has a dedicated
  endpoint (`blocksReorder(slug, ids)` in the SDK).
- **`block_type_id` is effectively dashboard-only knowledge.** The public read API
  returns each block's type *name* and its instance id — not its `block_type_id` — and
  the management plane has no read endpoints at all. To compose new blocks
  programmatically you must be given the block-type UUIDs (from the dashboard). If you
  don't have them, structure the content as **entries** instead (fully writable blind)
  and keep pages for dashboard-managed layout.
- MCP `create_page`/`update_page` extra caveats: the `title` argument is ignored by the
  backend (set `seo.title`), `seo` there accepts only `title`/`description`, and block
  `translations`/`seo_translations` aren't expressible.

## Practical write checklist

1. `get_content_type` / schema → exact field names, types, required, localizable, select options.
2. Uploads first → collect media UUIDs.
3. Compose `data` per the table above; localizable required fields go in `translations`.
4. Create (draft) → read back → verify → publish.
5. On `VALIDATION_ERROR` via MCP (no field detail): re-check step 1-3 against this file
   before retrying — the usual culprits are slug-instead-of-UUID in relation/image,
   markdown in richtext, and required-localizable fields sitting in `data`.
