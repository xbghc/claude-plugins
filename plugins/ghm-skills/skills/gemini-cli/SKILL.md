---
name: gemini-cli
description: Wield Google's Gemini CLI as an auxiliary tool for code review, web research, codebase analysis, and parallel tasks. Use when tasks benefit from a second AI perspective, current web information, or when user explicitly requests Gemini.
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
---

# Gemini CLI Integration Skill

Orchestrate Gemini CLI from Claude Code for code review, web research, architecture analysis, and parallel work.

## Command Pattern

```bash
gemini -p "prompt" --allowed-tools tool1,tool2 -o text 2>&1
```

Key flags:
- `-p "prompt"`: **Required for non-interactive (headless) mode**. Without `-p`, Gemini enters interactive mode and hangs.
- `--allowed-tools tool1,tool2`: Explicitly grant only the tools this task needs. Never use `--approval-mode=yolo`.
- `-o text`: Human-readable output (`json` for structured, `stream-json` for streaming)
- `-m MODEL`: Override model (usually unnecessary — model is configured in `~/.gemini/settings.json`)

## Gemini Tool Names

| Gemini tool | Purpose |
|-------------|---------|
| `read_file` | Read file contents |
| `write_file` | Create / overwrite files |
| `replace` | Edit files (find & replace) |
| `run_shell_command` | Execute shell commands |
| `glob` | Search files by name pattern |
| `grep_search` | Search file contents |
| `google_web_search` | Real-time Google search |
| `web_fetch` | Fetch URL content |
| `codebase_investigator` | Deep architectural analysis |
| `save_memory` | Cross-session persistent memory |

## When to Use

1. **Second opinion / cross-validation** — code review from a different AI perspective
2. **Google Search grounding** — current web info, latest versions, docs updates
3. **Codebase architecture** — Gemini's `codebase_investigator` tool
4. **Parallel processing** — offload tasks via background execution

When NOT to use: simple tasks (overhead not worth it), tasks needing immediate response, interactive refinement.

## File References

Use `@` to reference files in prompts:
```bash
gemini -p "Review @./src/auth.py and @./src/middleware.py" --allowed-tools read_file -o text 2>&1
```

## Background Execution

For long tasks, use Bash `run_in_background`:
```bash
gemini -p "Analyze entire codebase" --allowed-tools read_file,glob,grep_search -o text 2>&1
```

## Error Handling

- Rate limit (429): CLI auto-retries; if persistent, ask user whether to switch model
- Auth issues: check `gemini --version`, re-auth if needed
- Always validate Gemini's output before using
