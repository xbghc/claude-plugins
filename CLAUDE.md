# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a personal Claude Code **plugin marketplace** (`ghm-plugins`), containing multiple plugins that can be installed via `/plugin install <name>@ghm-plugins`.

## Architecture

```
.claude-plugin/marketplace.json   # Marketplace manifest — lists all plugins
plugins/
  stitch-ui/                      # Skills plugin (one skill per plugin)
    .claude-plugin/plugin.json
    CLAUDE.md                     # Maintenance rules for this plugin
    skills/stitch-ui/
      SKILL.md                    # Skill definition (triggered by Stitch-related requests)
      references/prompt-guide.md
  distill/                        # Skills plugin
    .claude-plugin/plugin.json
    skills/distill/SKILL.md
  gemini-cli/                     # Skills plugin
    .claude-plugin/plugin.json
    skills/gemini-cli/SKILL.md
  zotero-mcp/                     # MCP server plugin
    .claude-plugin/plugin.json
    .mcp.json                     # MCP server config (npx @xbghc/zotero-mcp)
  semanticscholar-mcp/            # MCP server plugin
    .claude-plugin/plugin.json
    .mcp.json                     # MCP server config (npx @xbghc/semanticscholar-mcp)
```

**Two types of plugins:**
- **Skills plugins** (`stitch-ui`, `distill`, `gemini-cli`): contain `skills/` with a single `SKILL.md`. Each skill is its own plugin so users can install them independently.
- **MCP server plugins** (`zotero-mcp`, `semanticscholar-mcp`, `telegram`): contain `.mcp.json` that defines the MCP server command and environment variables.

## Key Files

- **`marketplace.json`**: Central registry. Each entry has `name`, `description`, `source` (relative path), and `category`. Must be updated when adding/removing plugins.
- **`plugin.json`**: Per-plugin metadata (`name`, `version`, `description`, `keywords`). Keep these in sync with the plugin's actual contents.
- **`SKILL.md`**: Frontmatter (`name`, `description`) defines when the skill triggers. The `description` field is critical — it controls skill activation matching.
- **`.mcp.json`**: MCP server definition with `command`, `args`, and `env` (using `${VAR}` syntax for environment variables).

## Maintenance Rules

- Periodically sync `plugins/stitch-ui/skills/stitch-ui/references/prompt-guide.md` with the official Stitch prompting guide at https://stitch.withgoogle.com/docs/learn/prompting/.

## Adding a New Plugin

1. Create `plugins/<name>/` with `.claude-plugin/plugin.json`
2. Add either `skills/` (for skills) or `.mcp.json` (for MCP servers)
3. Add an entry to `.claude-plugin/marketplace.json`
4. Update `README.md`

## Environment Variables

- `zotero-mcp`: requires `ZOTERO_API_KEY`, `ZOTERO_USER_ID`
- `semanticscholar-mcp`: requires `SEMANTIC_SCHOLAR_API_KEY`
