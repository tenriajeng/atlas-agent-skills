# Atlas Agent Skills

Agent skill + MCP configuration for **[Atlas CMS](https://docs.atlas.latellu.com)** — teaches AI coding agents when and how to use the Atlas MCP tools, the [`@latellu/atlas-sdk`](https://www.npmjs.com/package/@latellu/atlas-sdk) delivery/management clients, and the [`@latellu/atlas-cli`](https://www.npmjs.com/package/@latellu/atlas-cli) type generator.

The skill itself is plain markdown ([`skills/atlas-cms/SKILL.md`](skills/atlas-cms/SKILL.md) plus per-surface references), so it works with **any** AI coding agent. This repo also packages it as a Claude Code plugin with the Atlas MCP server pre-wired.

## Claude Code

```bash
claude plugin marketplace add tenriajeng/atlas-agent-skills
claude plugin install atlas@latellu
```

Then set your credentials in the shell where you run `claude`:

```bash
export ATLAS_LIVE_API_KEY="atlas_live_xxx..."   # read tools
export ATLAS_MGMT_API_KEY="atlas_mgmt_xxx..."   # write tools
```

Create keys at [cms.atlas.latellu.com/dashboard/api-keys](https://cms.atlas.latellu.com/dashboard/api-keys). Full guide: [docs.atlas.latellu.com/docs/claude-code](https://docs.atlas.latellu.com/docs/claude-code).

## Other agents (Cursor, Codex, opencode, Gemini CLI, ...)

```bash
npm install -D @latellu/atlas-agent-skills
```

Then point your agent at the skill (via `AGENTS.md`, Cursor rules, etc.) and register the MCP server — see [`adapters/other-agents.md`](adapters/other-agents.md).

## What's inside

| Path | Purpose |
| --- | --- |
| `skills/atlas-cms/SKILL.md` | Router: shared Atlas concepts + decision table (MCP vs SDK vs CLI) |
| `skills/atlas-cms/references/` | Per-surface guides: MCP tools, SDK delivery, SDK management, CLI typegen |
| `.mcp.json` | Atlas MCP server config (used by the Claude Code plugin) |
| `.claude-plugin/` | Claude Code plugin + marketplace manifests |
| `adapters/other-agents.md` | Setup for non-Claude agents |

## Releasing

Bump `version` in `package.json` (and `.claude-plugin/plugin.json`), tag `vX.Y.Z`, push the tag — CI publishes to npm. The Claude Code marketplace resolves the plugin from npm, so users pick up the new version on `claude plugin marketplace update latellu`.

## License

[MIT](LICENSE)
