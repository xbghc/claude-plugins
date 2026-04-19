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

### verify-before-respond

Suppresses sycophantic agreement. Forces investigation (read code, check docs, verify facts) before agreeing, disagreeing, or implementing any user input. Generalizes superpowers' `receiving-code-review` and `verification-before-completion` patterns to all user inputs, not just code review or completion claims.

### jules-workflow

Delegate multi-file coding tasks to Google's Jules async agent. Pairs with the [`@xbghc/jules-cli`](https://www.npmjs.com/package/@xbghc/jules-cli) npm package for background `wait` — Claude is notified when the session hits a terminal state, without blocking the conversation.

## Installation

```
/plugin install stitch-ui@ghm-plugins
/plugin install distill@ghm-plugins
/plugin install gemini-cli@ghm-plugins
/plugin install academic-mcp@ghm-plugins
/plugin install verify-before-respond@ghm-plugins
/plugin install jules-workflow@ghm-plugins
```

## License

MIT
