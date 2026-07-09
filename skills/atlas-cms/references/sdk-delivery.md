# SDK delivery client (read)

```bash
npm install @latellu/atlas-sdk        # Node 18+ (uses global fetch)
```

## Canonical setup (Next.js thin adapter)

Keep one server-only adapter, e.g. `lib/atlas.ts`:

```ts
import "server-only";
import { createClient } from "@latellu/atlas-sdk";
import type { AtlasContentTypes } from "./atlas.types"; // generated — see cli-typegen.md

// The SDK sets NO cache directive. In Next.js you must decide caching yourself
// via a custom fetchImpl, or Next's defaults for un-annotated fetch apply.
const noStoreFetch: typeof fetch = (input, init) =>
  fetch(input, { ...init, cache: "no-store" }); // or next: { revalidate: 60 }

export const atlas = createClient<AtlasContentTypes>({
  url: process.env.ATLAS_BASE_URL ?? "",     // https://api.atlas.latellu.com
  apiKey: process.env.ATLAS_API_KEY ?? "",   // atlas_live_...
  fetchImpl: noStoreFetch,
});
```

With `no-store`, pair route files with `export const dynamic = "force-dynamic"` so
dashboard edits appear immediately. The SDK reads no env vars itself — `url`/`apiKey`
must be passed explicitly. `createClient` throws synchronously if either is missing.

## Surface

```ts
const { items, total, page, pageSize } =
  await atlas.entries("news-article").list({ locale: "en", page: 1, limit: 20, sort: "created_at:desc" });

const entry = await atlas.entries("news-article").get("hello-world", { locale: "en" }); // null on 404
// entry: { id, slug, status, published_at, data }  — data is already parsed to an object

const pages = await atlas.pages.list({ locale: "en", page: 1, limit: 20 }); // summaries, no blocks
const pageDetail = await atlas.pages.get("home", { locale: "en" });         // null on 404
// pageDetail: { id, slug, status, seo, blocks: [{ id, type, position, data }] }

const asset = await atlas.media.get(mediaId);      // null on 404
const raw = await atlas.raw.get<T>("/some/path");  // escape hatch for unwrapped /api/v1/public paths
```

Semantics the SDK handles for you (don't re-implement):
- `entry.data`, page `seo`, and block `data` arrive from the API as JSON strings — the
  SDK parses them.
- Locale merge is field-by-field: translated fields overlay base data; untranslated
  fields fall back to the base locale.
- Page blocks are always returned sorted by `position`, regardless of API order.

Errors: non-404 failures throw `AtlasError` (`{ status?, code?, message }`). `get()`
methods return `null` only for 404.

## Rendering blocks

Dispatch on `block.type` with an explicit fallback so new block types added in the
dashboard never crash the site:

```tsx
export function BlockRenderer({ blocks }: { blocks: AtlasBlock[] }) {
  return blocks.map((block) => {
    switch (block.type) {
      case "hero":   return <HeroBlock key={block.id} data={block.data} />;
      case "faq":    return <FaqBlock key={block.id} data={block.data} />;
      default:       return null; // or an <UnknownBlock /> placeholder
    }
  });
}
```

## Limits

- Draft preview is not yet supported by the SDK — delivery keys see published content only.
- There is no `search` or `schema` namespace on the client; schema introspection is the
  CLI's job (`cli-typegen.md`).
- `image` / `relation` / `content_type_reference` field values are raw string IDs/slugs —
  resolve with `atlas.media.get(id)` or `atlas.entries(type).get(slug)`.
