# atlas-cms skill evals

Eval suite for the `atlas-cms` skill ([../skills/atlas-cms/](../skills/atlas-cms/)).
Two eval types, run and fixed differently — see below.

This directory is dev-only: it's not in `package.json`'s `files` allowlist, so it never
ships in the published npm package.

## Eval types

- **`task-evals.json`** — task-execution correctness. Each case targets one specific
  documented gotcha (read-modify-write, blockless-publish, key-class mismatch, UUID vs
  slug, richtext format, pagination limits, retry semantics, etc.) in a specific
  reference file. `live: true` cases make real Atlas MCP tool calls; everything else is
  graded from the agent's produced code/plan/reasoning, no live calls.
- **`trigger-evals.json`** — does the skill's `description` frontmatter fire on the
  right prompts and stay quiet on near-misses (adjacent CMS/API-key/typegen requests
  that share keywords but need a different tool)?

## Running

### Env vars

Live cases in `task-evals.json` need a real Atlas management key:

```bash
export ATLAS_MGMT_API_KEY="atlas_mgmt_..."   # test/throwaway workspace only
```

**Never** put this key in `task-evals.json`, a prompt, or any committed file. It's
inherited from the shell by the subagent tool context that runs live cases, nothing
else. Saved transcripts in `results/` are redacted (`atlas_mgmt_[A-Za-z0-9_-]+` →
`[REDACTED]`) before being written, as a backstop.

Some cases are gated on `live_tier: "B"` (need `ATLAS_LIVE_API_KEY` too, for read
verification) — these are recorded `SKIPPED` (not `FAILED`) if that var is absent.

### Orchestration recipe (subagent-driven)

Model roles are fixed and intentional — Haiku is cheap enough to run per-case at scale
for execution/grading; Sonnet is reserved for the judgment-heavy work of actually
editing skill content:

1. **Trigger pass** — spawn one `model: "haiku"` subagent per query in
   `trigger-evals.json`, given only the raw query. Record whether it invokes the
   `atlas-cms` skill. Write `results/<run-id>/trigger-results.md`.
2. **Task-execution run** — per case in `task-evals.json`, two separate `model:
   "haiku"` subagents so grading isn't contaminated by the executor's own framing:
   - **Executor**: given the case prompt + read access to `skills/atlas-cms/**` (+ MCP
     tools with the inherited env key for `live: true` cases, with `cleanup.steps`
     framed as a mandatory final action regardless of earlier outcome). Saves a
     redacted transcript + tool results to `results/<run-id>/<case-id>/`.
   - **Grader**: fresh subagent, no shared context with the executor. Given the case's
     `assertions`, the saved transcript, and *only* the `reference_file` named in the
     case (not the whole skill). Writes `results/<run-id>/<case-id>/grading.json`
     (`{expectations: [{text, passed, evidence}], implicated_file, implicated_section}`
     on any failure).
   - The orchestrator (main session) independently verifies live-case cleanup from the
     saved tool results — see "Cleanup guardrail" below. Don't trust self-reports.
3. **Aggregate** all `grading.json` + `trigger-results.md` into
   `results/<run-id>/summary.md`.
4. **Fix loop** — group failing task-execution cases by `implicated_file`. For each
   file with ≥1 failure, spawn one `model: "sonnet"` subagent given only that file's
   current content + the failing assertions/evidence/implicated_section, scoped to
   patch only that file's implicated section(s) — not a full rewrite, not other files.
   Failing trigger queries get a `model: "sonnet"` subagent scoped to only the
   `description:` frontmatter line of `SKILL.md`.
5. **Rerun** only the previously-failed case/query ids (fresh Haiku executor+grader).
   Repeat steps 4–5 up to 3 rounds or until stable. A case failing identically twice in
   a row gets marked `needs-human-review` in `summary.md` instead of looping further.

## Cleanup guardrail (live cases)

Every live-created resource uses a `eval-test-<case-id>-<random>` slug (media: folder
`/eval-test/`, alt text prefixed `[eval-test]`) so it's trivially greppable and never
collides with real content. After each live case, the orchestrator parses its saved
tool results and confirms every `create_entry`/`create_page`/`upload_media` call has a
matching later `delete_*` call against the same id/slug with `deleted: true` in the
response. Missing cleanup **force-fails the case regardless of assertion outcome** and
surfaces the exact leftover slug/id in `summary.md` for manual removal via the
dashboard.

`mcp-bulk-delete-uuid-not-slug` additionally asserts its `bulk_delete_entries.ids`
contains exactly the UUIDs created earlier in the *same* transcript — never a
pattern-matched or `list_entries`-derived set — so the agent under test can't
generalize to bulk-deleting real workspace content.

## Results layout

```
results/
└── <run-id>/
    ├── trigger-results.md
    ├── summary.md
    └── <case-id>/
        ├── transcript.md       # redacted
        ├── tool-results.json   # redacted
        └── grading.json
```

`results/` is gitignored — raw transcripts may reflect real workspace content shapes
even after key redaction. Only `REPORT.md` (dated summaries) is committed.

## Fixtures

`fixtures/eval-test-pixel.png` — a 1×1 PNG used by `mcp-media-upload-validate` so that
case doesn't depend on an external file existing on the runner's machine.
