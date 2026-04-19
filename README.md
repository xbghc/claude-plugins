# ghm-plugins

Personal Claude Code plugin marketplace.

## Plugins

### stitch-ui

UI design with Stitch MCP. Two-phase delivery: generate screenshots for review, then produce code verified by Playwright.

### distill

Extract reusable lessons from session trial-and-error into skills or project docs.

### gemini-cli

Orchestrate Google's Gemini CLI from Claude Code for code review, web research, and parallel analysis.

### academic-mcp

Bundles two MCP servers for academic research:

- **zotero** — manage references and library (`@xbghc/zotero-mcp`)
- **semanticscholar** — paper search and retrieval (`@xbghc/semanticscholar-mcp`)

Requires env: `ZOTERO_API_KEY`, `ZOTERO_USER_ID`, `SEMANTIC_SCHOLAR_API_KEY`

## Installation

```
/plugin install stitch-ui@ghm-plugins
/plugin install distill@ghm-plugins
/plugin install gemini-cli@ghm-plugins
/plugin install academic-mcp@ghm-plugins
```

## License

MIT
