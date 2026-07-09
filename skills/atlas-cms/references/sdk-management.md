# SDK management client (write)

`@latellu/atlas-sdk/management` — Promise-based write client for `/api/v1/manage`.
**Server-side only.** Never import this module or expose an `atlas_mgmt_*` key in
browser-bundled code.

```ts
import { createManagementClient, AtlasError } from "@latellu/atlas-sdk/management";

const client = createManagementClient<AtlasContentTypes>({
  url: process.env.ATLAS_BASE_URL ?? "",    // https://api.atlas.latellu.com
  token: process.env.ATLAS_MGMT_KEY ?? "",  // MUST start with atlas_mgmt_
});
```

Construction rules:
- The token prefix is validated **synchronously**: a non-`atlas_mgmt_` token (e.g. a
  live key) throws `ManagementConfigError` immediately — wrap
  `createManagementClient(...)` in try/catch, not just the awaited calls.
- Requests carry `X-API-Key` and go to `${url}/api/v1/manage/*`.
- The key's scopes gate operations: `content:write` → create/update/delete/duplicate,
  `content:publish` → publish/unpublish/archive/schedule, `media:write` → uploads.
  Missing scope = 403 on that call only. Every write is audit-attributed to the key's
  creator and bounded by their RBAC — a key can never do more than its creator could in
  the dashboard.

## Entries

```ts
const w = client.entries("news-article");

await w.create({ slug: "hello", data: { title: "Hello", body: "..." } });
// CreateEntryInput = { slug, data, ... }  → creates a DRAFT
// POST /content-types/:type/entries
// Field value formats (richtext = Tiptap HTML, relation/image = UUIDs, ...):
//   see authoring.md before composing `data`.

await w.update("hello", { data: { title: "Hello v2", body: "..." } });
// UpdateEntryInput = { slug?, data?, ... } — send the FULL data object you want stored

// Localized content: pass a `translations` map as an extra key (accepted by the API
// on both create and update; required+localizable fields MUST live here, not in data):
await w.update("hello", {
  data: fullBaseData,
  translations: { de: { data: { title: "Hallo" } }, ja: { data: { title: "こんにちは" } } },
});

await w.publish("hello");                            // PATCH .../publish
await w.unpublish("hello");                          // PATCH .../unpublish
await w.archive("hello");                            // PATCH .../archive
await w.schedule("hello", "2026-08-01T09:00:00Z");   // PATCH .../schedule { publish_at } — ISO 8601 UTC
await w.duplicate("hello");                          // POST .../duplicate → new draft copy
await w.delete("hello");                             // DELETE
await w.bulk([                                       // POST .../bulk { operations }
  { op: "publish", id: "…" },
  { op: "archive", id: "…" },
]);
```

All idOrSlug parameters accept either the entry ID or its slug.

## Pages

```ts
// NOTE: pages have NO top-level title — the title lives in seo.title.
// Block objects need a block_type_id UUID; see authoring.md for the full write shape
// (seo_translations, block translations, nesting) and the block_type_id caveat.
await client.pages.create({ slug: "landing", seo: { title: "Landing" }, blocks: [...] });
await client.pages.update("landing", { seo: { title: "Landing v2" } });
await client.pages.publish("landing");   // fails unless the page has ≥1 block — see authoring.md
await client.pages.unpublish("landing");   // pages have the FULL lifecycle here
await client.pages.archive("landing");     //   (unlike the MCP tools)
await client.pages.schedule("landing", "2026-08-01T09:00:00Z");
await client.pages.delete("landing");
await client.pages.blocksReorder("landing", ["block-id-2", "block-id-1", "block-id-3"]);
// PATCH /pages/:slug/blocks/reorder { block_ids } — pass the COMPLETE ordered id list
```

## Media

```ts
import { readFile } from "node:fs/promises";

const blob = new Blob([await readFile("./team.jpg")], { type: "image/jpeg" });
const asset = await client.media.upload(blob, { alt: "Team photo" });
// multipart POST /media/upload; returns UploadedMediaAsset = { id, ... }

await client.media.delete(asset.id);
```

Server-side upload validation: max 10 MB; MIME must be `image/*`, `video/*`, or
`application/pdf` — set the Blob `type` explicitly, an untyped Blob is rejected.

## Idempotency & retries (built in)

- Every write accepts an optional trailing options argument
  `{ idempotencyKey: "stable-string" }` → sent as the `Idempotency-Key` header. Pass a
  **stable** key when your own retry logic must not double-create or double-publish:

  ```ts
  await w.create(input, { idempotencyKey: `import-article-${sourceId}` });
  ```

- **429 responses are retried automatically**: exponential backoff from 200 ms, up to 3
  retries. Hardcoded — not configurable, and only for 429 (400/403/500 are not retried).

## Error handling

```ts
try {
  await w.create({ slug: "x", data: { title: 42 } });
} catch (e) {
  if (e instanceof AtlasError) {
    e.status;  // 400
    e.code;    // "VALIDATION_ERROR"
    e.errors;  // [{ field: "title", message: "must be a string" }, ...]  ← per-field detail
  }            //    (structured here; MCP ≥ 1.1.0 appends it to the error text) — surface it to users
}
```

`ManagementConfigError` (bad config, thrown synchronously) is a different class from
`AtlasError` (failed requests, rejected promises) — handle both.

## Worked example: idempotent import script

```ts
for (const row of sourceRows) {
  const slug = slugify(row.title);
  const existing = await readClient.entries("article").get(slug); // delivery client
  const w = client.entries("article");
  if (existing) {
    await w.update(slug, { data: mapRow(row) });
  } else {
    await w.create({ slug, data: mapRow(row) }, { idempotencyKey: `import-${row.id}` });
  }
  await w.publish(slug);
}
```

## Effect-native variant

`@latellu/atlas-sdk/management/effect` exports `makeManagementClient` — identical
surface where every method returns `Effect.Effect<T, AtlasError>`; run with
`Effect.runPromise`. The plain entrypoints never bundle `effect` as a dependency; only
this subpath does. Gotcha if you compose Effects yourself: `Effect.runPromise` rejects
with a `FiberFailure` wrapper, not the raw error — recover the original `AtlasError` via
`Runtime.isFiberFailure` + `Cause.squash` (the Promise surface already does this
internally).
