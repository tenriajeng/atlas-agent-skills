# SDK management client (write)

Server-side only. Never import this module or expose an `atlas_mgmt_*` key in
browser-bundled code.

```ts
import { createManagementClient, AtlasError } from "@latellu/atlas-sdk/management";

const client = createManagementClient<AtlasContentTypes>({
  url: process.env.ATLAS_BASE_URL ?? "",    // https://api.atlas.latellu.com
  token: process.env.ATLAS_MGMT_KEY ?? "",  // MUST start with atlas_mgmt_
});
```

`createManagementClient` throws `ManagementConfigError` **synchronously** if the token
doesn't start with `atlas_mgmt_` (e.g. you passed a live key) — wrap construction in
try/catch, not just the calls. Requests go to `/api/v1/manage` with `X-API-Key`.

## Surface

```ts
const w = client.entries("news-article");
await w.create({ slug: "hello", data: { title: "Hello" } });
await w.update("hello", { data: { title: "Hello v2" } });
await w.publish("hello");            // needs content:publish scope
await w.unpublish("hello");
await w.archive("hello");
await w.schedule("hello", "2026-08-01T09:00:00Z");
await w.duplicate("hello");
await w.delete("hello");
await w.bulk([{ op: "publish", id: "..." }, ...]);

await client.pages.create({ slug: "landing", /* ... */ });
await client.pages.update("landing", { /* ... */ });
await client.pages.publish("landing");     // also: unpublish, archive, schedule, delete
await client.pages.blocksReorder("landing", ["block-id-1", "block-id-2"]);

const asset = await client.media.upload(fileBlob, { alt: "Team photo" }); // multipart
await client.media.delete(asset.id);
```

Every write accepts an optional trailing `{ idempotencyKey: "..." }` — pass a stable key
when a retry must not double-create/double-publish.

## Built-ins

- **429s are retried automatically** (exponential backoff from 200 ms, up to 3 retries).
  Not configurable.
- **Validation errors carry per-field detail** (unlike MCP): `AtlasError.errors` is
  `{ field, message }[]` on 400s — surface it to users instead of the generic message.
- Scopes: `content:write` covers create/update/delete/duplicate; `content:publish` covers
  publish/unpublish/archive/schedule; `media:write` covers uploads. Missing scope → 403.

## Effect-native variant

`@latellu/atlas-sdk/management/effect` exports `makeManagementClient` with the same shape
where every method returns `Effect.Effect<T, AtlasError>`. The plain Promise client above
never bundles `effect`. If you run Effects yourself, note `Effect.runPromise` rejects
with a `FiberFailure` wrapper — recover the original `AtlasError` via
`Runtime.isFiberFailure` + `Cause.squash` (the Promise surface already does this for you).
