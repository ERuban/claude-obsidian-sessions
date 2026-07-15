---
context: conversation
description: Save current session to Obsidian vault as structured session log
model: sonnet
allowed-tools: Read, Write, Bash, Glob, Edit, AskUserQuestion
---

# Compress Session to Obsidian Vault

Save the current conversation as a structured, searchable log in the Obsidian vault named `${user_config.vault_name}`.

## Step 0: Resolve the vault path

Two config values control this: `${user_config.vault_path}` (optional override) and `${user_config.vault_name}`.

1. **If `${user_config.vault_path}` is set** — non-empty AND not a literal unexpanded placeholder (when the option is not configured, it renders as the raw `${user_config...}` text: treat that as NOT set, never as a path) — that value is the vault base, call it `{vault}` (expand a leading `~`). Obsidian's registry is NOT consulted, so this works on machines without Obsidian. If the folder doesn't exist yet, create it with `mkdir -p` — the user pointed the plugin there deliberately.
2. **Otherwise** resolve it from Obsidian's vault registry by matching the vault name `${user_config.vault_name}`:
   1. Read Obsidian's config file (pick the one for the current OS):
      - macOS: `~/Library/Application Support/obsidian/obsidian.json`
      - Linux: `~/.config/obsidian/obsidian.json`
      - Windows: `%APPDATA%\obsidian\obsidian.json`
   2. It contains a `vaults` map: `{ "<id>": { "path": "/abs/path/to/vault", ... }, ... }`. Find the entry whose folder name (last path segment) equals `${user_config.vault_name}`. That `path` is the vault base — call it `{vault}`.
   3. If no vault matches, stop and tell the user the vault name wasn't found in Obsidian's registry, and list the vault folder names that ARE present so they can fix the config (or set the `vault_path` override).

`{vault}` contains `Projects/` (and possibly `Templates/`). Session logs live under `{vault}/Projects/{project}/Sessions/`.

## Step 1: Detect project & locate THIS session's log

1. Run `pwd`; project name = last path segment of the current directory.
2. Map to a folder under `{vault}/Projects/`. Glob `Projects/*/` and match by name (exact or closest). If none matches, use the project name verbatim and create `Projects/{project}/Sessions/` (`mkdir -p`). Creating a missing `Sessions/` folder is the ONLY filesystem change allowed here — never delete or overwrite anything else in the vault.
3. **One log file per Claude session — never duplicate, never clobber others.**
   - If you already wrote a session log earlier in THIS conversation, update that exact file. You know its path from this conversation — reuse it; do NOT generate a new timestamp/filename.
   - Otherwise this is the first `/compress` of the session: create a new log now and remember its path for the rest of the conversation.
   - Updating = read the file, merge new content into existing sections, append to Raw Session Log, keep the original filename and time. Never truncate it and never touch any OTHER session file.

## Step 2: One question

Single `AskUserQuestion` call, two fields:
- **Topic** — suggest 2-3 hyphenated names from the conversation (e.g. `auth-middleware-refactor`); user picks or types own. (On update: reuse existing topic, don't re-ask.)
- **Notes** — "Any extra notes? (skip to skip)".

## Step 3: Write the log

Auto-select which sections to include based on what actually happened in the session — omit empty ones. Quick Reference, Quick Resume Context, and Raw Session Log are always included.

Extract keywords from the conversation: services/repos, frameworks/libraries, action types (refactor/fix/setup/debug/implement/test), tools (Docker/PHP/Symfony/AWS…), identifiers (ticket IDs, PR numbers, branch), people.

Model: use the short human-readable form of the current model (e.g. `opus 4.8 (1M)`, `sonnet 4.6`). You are told the model in system context.

Filename: `{YYYY-MM-DD}-{HH_MM}-{topic}.md` → `{vault}/Projects/{folder}/Sessions/`.

```markdown
---
type: session
date: {YYYY-MM-DD}
time: "{HH:MM}"
project: {project-name}
topic: {topic-name}
model: "{model-name}"
tags: [{keywords}]
status: completed
---

# Session: {DD-MM-YYYY} {HH:MM} — {Topic Name}

## Quick Reference
**Topics:** {comma-separated keywords}
**Project:** {project-name}
**Outcome:** {1-2 sentence result}
**Model:** {model-name}
**Branch:** {git branch if any}
**Ticket:** {ticket ID if any}

## Decisions Made
{decisions with rationale — only if any}

## Key Learnings
{new understanding — only if any}

## Solutions & Fixes
{problem → solution — only if any}

## Files Modified
{path — what changed — only if any}

## Setup & Config
{env/config changes — only if any}

## Pending Tasks
- [ ] {remaining work — only if any}

## Errors & Workarounds
{errors and how handled — only if any}

## Custom Notes
{user notes from Step 2 — only if provided}

## Quick Resume Context
{2-3 sentences optimized for /resume to understand this session fast}

---

## Raw Session Log
{Thorough summary of the full conversation — key code, decisions, errors, solutions.
A summary, not a literal copy. NEVER read by /resume; kept for full-text search.}
```

## Step 4: Confirm

```
Session saved: Projects/{folder}/Sessions/{filename}
Sections: {list}
Keywords: {keywords}
Resume: /resume {topic}
```

## Rules

- **Privacy & scope**: only ever write inside the resolved vault path above. Never read, list, or reference any other Obsidian vault. Never record personal directory paths, usernames, e-mails, API tokens — substitute `<vault>`/`<user>` or omit.
- Frontmatter must be valid YAML — quote strings containing colons, tags as an array.
- Filename date: `YYYY-MM-DD` (sorting). Heading date: `DD-MM-YYYY` (readability).
- On update: preserve the original time in the filename; merge, don't overwrite.
- If run mid-session, note "Session ongoing" in Outcome.
