# SDK delivery client (read)

`@latellu/atlas-sdk` — typed, zero-dependency read client for `/api/v1/public`.
Node 18+ (uses global `fetch`). ESM only.

```bash
npm install @latellu/atlas-sdk
```

## Client setup

```ts
import { createClient } from "@latellu/atlas-sdk";

const atlas = createClient<AtlasContentTypes>({
  url: "https://api.atlas.latellu.com", // required — SDK reads NO env vars itself
  apiKey: "atlas_live_...",             // required — delivery key
  fetchImpl: customFetch,               // optional — see "Caching" below
});
```

- `createClient` **throws synchronously** (`AtlasError`) if `url` or `apiKey` is missing.
- The generic `<AtlasContentTypes>` comes from CLI codegen (`cli-typegen.md`); without
  it, `entry.data` is `Record<string, unknown>`.
- All requests go to `${url}/api/v1/public/*` with header `X-API-Key`.

### Canonical Next.js thin adapter

Keep one server-only adapter (pattern used by all official Atlas examples), e.g.
`lib/atlas.ts`:

```ts
import "server-only";
import { createClient } from "@latellu/atlas-sdk";
import type { AtlasContentTypes } from "./atlas.types";

const noStoreFetch: typeof fetch = (input, init) =>
  fetch(input, { ...init, cache: "no-store" });

export const atlas = createClient<AtlasContentTypes>({
  url: process.env.ATLAS_BASE_URL ?? "",
  apiKey: process.env.ATLAS_API_KEY ?? "",
  fetchImpl: noStoreFetch,
});
```

### Caching — the SDK deliberately has no knob

The SDK issues plain `fetch(url, { headers })` with **no cache directive**. In Next.js
App Router this means caching behavior is whatever Next's defaults do for un-annotated
fetch (varies by version/runtime). Decide explicitly via `fetchImpl`:

| Goal | fetchImpl |
|---|---|
| Dashboard edits appear immediately (no SSG) | `cache: "no-store"` + `export const dynamic = "force-dynamic"` in routes |
| ISR-style freshness | `next: { revalidate: 60 }` |
| Astro/SSR outside Next | default fetch is fine (no framework cache layer) |

## Full surface

```ts
interface AtlasClient<TSchema> {
  entries<K extends keyof TSchema>(type: K): EntriesResource<TSchema[K]>;
  pages: PagesResource;
  media: MediaResource;
  raw: Requester;  // low-level escape hatch
}
```

### Entries

```ts
const res = await atlas.entries("news-article").list({
  locale: "en",              // optional
  page: 1, limit: 20,        // offset pagination (limit max 100, page max 1000)
  sort: "created_at:desc",   // "<field>:<asc|desc>", e.g. "published_at:desc"
});
// res: ListResult<AtlasEntry<T>> = { items, total, page, pageSize }

const entry = await atlas.entries("news-article").get("hello-world", { locale: "en" });
// AtlasEntry<T> = { id, slug, status, published_at: string | null, data: T }
// null on 404; throws AtlasError on anything else
```

What the SDK does for you here (do not re-implement):
- `data` arrives from the API as a **JSON string**; the SDK parses it to an object
  (empty string → `{}`).
- With `locale`, `get()` merges the matching `translations[].data` over the base `data`
  field-by-field (`{ ...base, ...translation }`) — untranslated fields keep base values.

### Pages

```ts
const list = await atlas.pages.list({ locale: "en", page: 1, limit: 20 });
// items: AtlasPageSummary[] — lightweight rows { id, slug, status, ... }, NO blocks

const page = await atlas.pages.get("home", { locale: "en" }); // null on 404
// AtlasPage = { id, slug, status, seo: AtlasPageSEO, blocks: AtlasBlock[] }
// AtlasPageSEO = { title?, description?, keywords?, og_image?, canonical?, ... }
// AtlasBlock  = { id, type, position, data: Record<string, unknown> }
```

`pages.get` semantics: SEO is overlaid with `seo_translations` for the locale
(field-level fallback); each block's `data` is overlaid with its matching
`block_translations` row; **blocks are always returned sorted by `position`** regardless
of API order; `seo`/block `data` JSON strings are parsed defensively (malformed JSON
silently becomes `{}` — a corrupt block fails silent, not loud).

### Media & raw

```ts
const asset = await atlas.media.get(id);   // MediaAsset = { id, ...unknown } | null on 404

const { data, meta } = await atlas.raw.get<T>("/entries", { type: "article", cursor });
// any /api/v1/public path; use for things the SDK doesn't wrap — e.g. CURSOR pagination
// (entries.list only exposes page/limit; past page 1000 the API returns PAGE_TOO_DEEP,
//  so deep datasets must iterate meta.next_cursor via raw.get)
```

## Error handling

```ts
import { AtlasError } from "@latellu/atlas-sdk";

try {
  const items = await atlas.entries("article").list();
} catch (e) {
  if (e instanceof AtlasError) {
    e.status; // HTTP status (undefined for network errors)
    e.code;   // "UNAUTHORIZED" | "FORBIDDEN" | "PAGE_TOO_DEEP" | ...
  }
}
```

Only `get()` methods special-case 404 → `null`. `list()` never returns null. Network
failures are wrapped in `AtlasError` with the URL in the message.

## Rendering blocks

Dispatch on `block.type` with an explicit fallback so block types added later in the
dashboard never crash the site — adding a section type then only requires registering a
component here:

```tsx
export function BlockRenderer({ blocks }: { blocks: AtlasBlock[] }) {
  return blocks.map((block) => {
    switch (block.type) {
      case "hero": return <HeroBlock key={block.id} data={block.data} />;
      case "faq":  return <FaqBlock key={block.id} data={block.data} />;
      default:     return <UnknownBlock key={block.id} type={block.type} />; // never throw
    }
  });
}
```

## Resolving reference fields

`image` / `relation` / `content_type_reference` values are raw strings, never resolved
objects. Content written through the API holds **UUIDs** (that's what the backend
validates); imported/seeded datasets may hold URLs instead — branch on the shape:

```ts
const article = await atlas.entries("article").get(slug);

const cover = article.data.cover_image.startsWith("http")
  ? article.data.cover_image                       // legacy/imported: direct URL
  : await atlas.media.get(article.data.cover_image); // API-written: media UUID

// relation fields hold entry UUIDs; the public API fetches entries by SLUG only,
// so resolve relations by listing the target type and matching on id:
const authors = await atlas.entries("author").list({ limit: 100 });
const author = authors.items.find((a) => a.id === article.data.author);
```

Also note: list endpoints have **no field-value filters** (only type/locale/page/sort) —
"entries where category = X" means paging + filtering client-side.

## Limits to remember

- Delivery keys see **published content only**; draft preview is not yet supported by
  the SDK (preview-token endpoint pending).
- No `search` or `schema` namespace on the client — schema introspection belongs to the
  CLI (`cli-typegen.md`); anything else goes through `raw.get`.
- API keys are rate limited; a 429 here is **not** retried automatically (only the
  management client retries) — back off exponentially and retry yourself.
