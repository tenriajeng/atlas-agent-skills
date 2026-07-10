# Atlas Agent Skills

Agent skill + MCP configuration for **[Atlas CMS](https://docs.atlas.latellu.com)** — teaches AI coding agents when and how to use the Atlas MCP tools, the [`@latellu/atlas-sdk`](https://www.npmjs.com/package/@latellu/atlas-sdk) delivery/management clients, and the [`@latellu/atlas-cli`](https://www.npmjs.com/package/@latellu/atlas-cli) type generator.

The skill itself is plain markdown ([`skills/atlas-cms/SKILL.md`](skills/atlas-cms/SKILL.md) plus per-surface references), so it works with **any** AI coding agent. This repo also packages it as a Claude Code plugin with the Atlas MCP server pre-wired.

**New to MCP?** MCP (Model Context Protocol) is just the plug-in standard that lets your AI agent call a tool directly — installing "the Atlas MCP server" gives your agent tools like "list entries" or "publish page" it can call on its own, instead of you copy-pasting API calls for it.

## Claude Code

```bash
claude plugin marketplace add tenriajeng/atlas-agent-skills
claude plugin install atlas@latellu
```

### Get your API keys

Atlas uses two separate keys so a read-only integration can never accidentally write:

- **`atlas_live_...`** — read-only. Fetches published content (blog posts, pages, media). Use this if your agent only needs to *display* content.
- **`atlas_mgmt_...`** — read/write. Lets your agent create, edit, and publish content. Only create this if you want your agent to make changes for you.

To create them: go to [cms.atlas.latellu.com/dashboard/api-keys](https://cms.atlas.latellu.com/dashboard/api-keys), log in to (or create) your workspace, click **New API key**, pick the key type above, and copy the value shown — it's only shown once.

Then set them in the shell where you run `claude`:

```bash
export ATLAS_LIVE_API_KEY="atlas_live_xxx..."   # read tools
export ATLAS_MGMT_API_KEY="atlas_mgmt_xxx..."   # write tools
```

Add these two `export` lines to your shell profile (`~/.zshrc` or `~/.bashrc`) if you want them available in every new terminal, not just the current session.

### Verify it worked

Start `claude` in your project and ask it something like *"list the content types in my Atlas workspace"*. If it calls an Atlas tool and returns a real answer, setup is complete. If it says it has no Atlas tools available, double check the plugin installed (`claude plugin install atlas@latellu` again) and that both env vars are set in that same shell (`echo $ATLAS_LIVE_API_KEY`).

Full guide: [docs.atlas.latellu.com/docs/agent-skill](https://docs.atlas.latellu.com/docs/agent-skill).

## Other agents (Cursor, Codex, opencode, Gemini CLI, ...)

```bash
npm install -D @latellu/atlas-agent-skills
```

Then point your agent at the skill (via `AGENTS.md`, Cursor rules, etc.) and register the MCP server — see [`adapters/other-agents.md`](adapters/other-agents.md).

## What's inside

| Path | Purpose |
| --- | --- |
| `skills/atlas-cms/SKILL.md` | Router: shared Atlas concepts + decision table (MCP vs SDK vs CLI) |
| `skills/atlas-cms/references/` | Per-surface guides: MCP tools, SDK delivery, SDK management, CLI typegen, content authoring (field formats, translations, blocks) |
| `.mcp.json` | Atlas MCP server config (used by the Claude Code plugin) |
| `.claude-plugin/` | Claude Code plugin + marketplace manifests |
| `adapters/other-agents.md` | Setup for non-Claude agents |

## Releasing (manual)

1. Bump `version` in `package.json` **and** `.claude-plugin/plugin.json` (keep them equal)
2. `npm publish` (from an authenticated machine)
3. `git tag vX.Y.Z && git push && git push --tags`

The Claude Code marketplace resolves the plugin from npm, so users pick up the new version on `claude plugin marketplace update latellu`.

## License

[MIT](LICENSE)
