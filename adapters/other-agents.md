# Using the Atlas skill outside Claude Code

> New to Atlas or MCP? Read the [README](../README.md) first — it explains what MCP is
> and the difference between `atlas_live_` (read-only) and `atlas_mgmt_` (read/write)
> keys before you get here.

The skill is plain markdown (`skills/atlas-cms/SKILL.md` + `references/`), so any AI
coding agent can consume it. Install the package into your project, then point your
agent at it.

```bash
npm install -D @latellu/atlas-agent-skills
```

## AGENTS.md-based agents (Codex, opencode, Gemini CLI, Jules, Amp, ...)

Add to your project's `AGENTS.md`:

```markdown
## Atlas CMS

When a task involves Atlas CMS (entries, pages, media, @latellu/atlas-sdk,
@latellu/atlas-cli, atlas_live_/atlas_mgmt_ keys, or api.atlas.latellu.com), first read
node_modules/@latellu/atlas-agent-skills/skills/atlas-cms/SKILL.md and follow its
routing table — it links to detailed references per surface (MCP, SDK read, SDK write,
CLI type generation). Do not guess Atlas API shapes from memory.
```

## Cursor

Create `.cursor/rules/atlas-cms.mdc`:

```markdown
---
description: Atlas CMS integration guidance (SDK, CLI, MCP)
alwaysApply: false
---

When working with Atlas CMS, read
node_modules/@latellu/atlas-agent-skills/skills/atlas-cms/SKILL.md and the reference
file it routes you to.
```

## MCP server (any MCP-capable agent)

"Registering an MCP server" means telling your agent to launch this process and use the
tools it exposes (list entries, publish page, etc.) — it's the same two API keys as
above, just read by a different process. The Atlas MCP server is a separate npm package
(`@latellu/atlas-mcp`, stdio). Register it with your agent's MCP configuration:

```json
{
  "mcpServers": {
    "atlas": {
      "command": "npx",
      "args": ["-y", "@latellu/atlas-mcp"],
      "env": {
        "ATLAS_LIVE_API_KEY": "atlas_live_xxx...",
        "ATLAS_MGMT_API_KEY": "atlas_mgmt_xxx..."
      }
    }
  }
}
```

- Cursor: `~/.cursor/mcp.json` or `.cursor/mcp.json`
- Codex CLI: `~/.codex/config.toml` (`[mcp_servers.atlas]`)
- opencode: `opencode.json` (`"mcp"` section)
- Others: consult your agent's MCP docs; the command/env above are the same everywhere.

See [docs.atlas.latellu.com/docs/mcp](https://docs.atlas.latellu.com/docs/mcp) for the
tool catalog and key setup.
