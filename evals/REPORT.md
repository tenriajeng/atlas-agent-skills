# Eval run history

Dated log of eval suite runs — pass rates, notable regressions/fixes, and harness
verification results. See [README.md](README.md) for how to run the suite.

## 2026-07-10 — run-20260710-191633 (initial suite)

First eval suite for `atlas-cms` (previously zero coverage). 25 task-execution cases + 14
trigger-accuracy queries, subagent-driven (Haiku executors/graders/harness-checks, Sonnet
fixers). Full detail: [results/run-20260710-191633/summary.md](results/run-20260710-191633/summary.md).

- **Task-execution: 22/25 fully passed**, 2 real defects found + fixed, 3 environment-blocked
  (live test-workspace key mismatch, not a skill defect).
- **Trigger-accuracy: 71% → 93%** after a description fix; 1 query (`t03`) flagged
  `needs-human-review` — a structural gap (direct MCP tool availability bypasses skill
  triggering) that no description wording can close.
- **Fixes shipped:**
  - `skills/atlas-cms/references/sdk-management.md` — `update()` example now leads with
    read-modify-write as a stated MUST, not an optional aside.
  - `skills/atlas-cms/SKILL.md` — added a bright-line MCP-vs-SDK routing rule (scale/
    recoverability, not run-count) and made the `description` frontmatter more insistent
    for casually-phrased requests.
- **Harness verification**: cleanup guardrail and trigger description-sensitivity both
  confirmed working via independent synthetic checks; a 3-file mutation test found 2/3
  removed gotchas didn't break agent behavior (redundant doc coverage / sound default
  engineering judgment), 1/3 did (UUID-vs-slug on relation fields) — informs future doc
  priority, not actioned this round.

<!-- Append new entries above this line, most recent first. -->
