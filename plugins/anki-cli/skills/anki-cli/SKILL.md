---
name: anki-cli
description: Use the `anki` CLI (anki-cli package) to manage Anki flashcards from the terminal — headless sync with AnkiWeb, query decks/notes/cards as JSON, create/update/delete notes, and rate cards to advance the SRS scheduler. Trigger on mentions of Anki, flashcards, spaced repetition, `anki-cli`, or requests to read/add/edit Anki cards programmatically.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# anki-cli

`anki` (the `anki-cli` package) is a headless Anki sync client with JSON I/O. It runs on servers and in agent sessions where the Anki desktop app cannot — it does not need Anki GUI, only the user's AnkiWeb account.

Source / install: https://github.com/xbghc/anki-cli

## When to use

- Ingest learning material → create new Anki notes in bulk.
- Query the user's collection (decks, notetypes, notes, cards, scheduling state).
- Update note fields or tags (e.g., fix typos, add metadata, retag in bulk).
- Delete obsolete notes.
- Record review outcomes via `answer-card` — the card's next review date is computed by Anki's FSRS / SM-2 scheduler.

**Not suitable for:** viewing card renderings (HTML), importing media files, managing deck configs. Those belong in the Anki GUI or AnkiWeb.

## First-run setup

If `anki --version` fails, install:

```bash
uv tool install git+https://github.com/xbghc/anki-cli
```

Then log in and pull the collection once:

```bash
anki login -u EMAIL                # prompts for password
anki sync                          # first sync does a full download
```

Credentials are stored in `~/.config/anki-cli/config.json` (0600); the collection is cached at `~/.local/share/anki-cli/collection.anki2`. Both paths are persistent — you only set up once per machine.

### Headless / long-running agent setup

For agents that run without a terminal (pm2, cron, server bots), the interactive password prompt will hang. Two options:

1. **Env vars** (recommended): set `ANKIWEB_USERNAME` and `ANKIWEB_PASSWORD` in the process environment, then run `anki login` once during first deploy. The CLI auto-reads them when flags are absent. Hostkey persists afterward, so the env vars are only needed for the one-time login — you can even remove them from `.env` after the first successful login if you prefer.
2. **Pre-login on a dev machine**, then copy `~/.config/anki-cli/config.json` to the server.

After either, call `anki sync` at least once so `~/.local/share/anki-cli/collection.anki2` exists.

## Commands

Run `anki <cmd> --help` for flags; all commands output JSON on stdout.

**Read-only:**

- `anki decks` — list decks
- `anki notetypes` — list notetypes (with field names)
- `anki notes "QUERY" [--limit N]` — search notes with [Anki query syntax](https://docs.ankiweb.net/searching.html)
- `anki note NOTE_ID` — one note
- `anki cards "QUERY" [--limit N]` — search cards (includes scheduling state)
- `anki card CARD_ID` — one card

**Write (affect local collection only until next `anki sync`):**

- `anki add-note` — reads JSON from stdin: `{"deck": "...", "notetype": "...", "fields": {...}, "tags": [...]}`
- `anki update-note NOTE_ID` — reads JSON from stdin: `{"fields": {...}, "tags": [...]}` (fields merge, tags replace)
- `anki delete-note NOTE_ID` — remove note + its cards
- `anki answer-card CARD_ID {again|hard|good|easy}` — rate a card; SRS scheduler advances it

## Sync discipline

The CLI is **explicit-sync**: commands never sync implicitly. Manage the cadence by task, not by session:

1. **Before a block of anki work**: `anki sync` — pull so you see what the user did on desktop/mobile since last time.
2. **Do the work** — queries / writes, in any order.
3. **After the block**: `anki sync` — push so the user's next device sync sees your changes.

A "block" is whatever coherent set of operations the current user turn implies. For interactive chat, one sync before any anki tool calls and one after is plenty. For autonomous / scheduled jobs (cron-triggered review summaries, proactive chats), sync at both ends of the job — the user may edit on their phone between runs.

Skipping the final sync is the #1 mistake: writes stay local and AnkiWeb (phone, desktop) never sees them.

## Typical workflows

**Distill notes from chat material into flashcards:**

```bash
anki sync
anki notetypes                       # confirm field names for target notetype
anki add-note <<'EOF'
{"deck":"Languages::Japanese","notetype":"Basic","fields":{"Front":"食べる","Back":"to eat (ichidan verb)"},"tags":["verb","frequency-1000"]}
EOF
anki sync
```

**Audit due cards, record a review:**

```bash
anki cards "deck:Japanese is:due" --limit 5   # pick one due card
anki answer-card 1234567890 good              # rate it
anki sync                                      # push rating to AnkiWeb
```

**Correct a field in an existing note:**

```bash
anki notes "front:typoed" --limit 1
# agent reads target note ID from output, then:
echo '{"fields":{"Front":"fixed spelling"}}' | anki update-note 1234567890
anki sync
```

## Pitfalls

- **Forgetting the final `anki sync`.** Writes are only local; without sync, AnkiWeb (and the user's phone/desktop) never sees them.
- **Notetype field-name mismatch.** Field names are case-sensitive and must match the notetype exactly. If unsure, run `anki notetypes` first.
- **Deck must already exist** for `add-note`. The CLI doesn't auto-create decks; ask the user to create the target deck in Anki if it doesn't exist.
- **`answer-card` is effectively irreversible.** Rating a card advances its scheduler state; there is no clean undo. Only call `answer-card` when the user explicitly asks you to record a review.
- **`FULL_UPLOAD` / `FULL_SYNC` errors from `sync`.** These mean local and remote have diverged in an unsafe way. Do not try to force-resolve; report to the user and let them fix it in the Anki desktop app.
