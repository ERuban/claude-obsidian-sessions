---
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

1. Detect the project name. Run:
   ```bash
   git rev-parse --path-format=absolute --git-common-dir 2>/dev/null
   ```
   If it prints a path, the project name is the name of that path's parent folder (the repository root — the same from any subfolder or git worktree). If it prints nothing (not a git repo), the project name is the last path segment of `pwd`.
2. Map to a folder under `{vault}/Projects/`. Glob `Projects/*/` and match by name (exact or closest). If none matches, use the project name verbatim and scaffold it: create `Projects/{project}/Sessions/` (session logs) and `Projects/{project}/Ready-to-Dev/` (backlog & decisions deferred during other work, one `type: task` file each — see `Templates/Task.md`) via `mkdir -p`. Also seed missing vault-level files, each ONLY if absent: `{vault}/Templates/Session.md`, `Templates/Task.md` and `{vault}/Home.md`, copied from the plugin's bundled templates at `${CLAUDE_PLUGIN_ROOT}/templates/` (if that renders as a literal unexpanded placeholder, use the newest `~/.claude/plugins/cache/obsidian-tools/obsidian/*/templates/` instead). These create-only writes are the ONLY filesystem changes allowed here — never delete or overwrite anything in the vault.
3. **One log file per piece of work — never duplicate, never clobber others.** Three cases:
   - **Same conversation, already saved earlier**: update that exact file. You know its path from this conversation — reuse it; do NOT generate a new timestamp/filename.
   - **New conversation continuing an existing topic**: if `Sessions/` already holds a log for the same work (same ticket ID or topic slug), offer it in Step 2. If the user picks it, append a `## Follow-up — {YYYY-MM-DD}` section (before `## Quick Resume Context`), refresh `updated`, `status`, `outcome`, Quick Resume Context and Pending Tasks to the latest state, and append to Raw Session Log. Keep the original filename, `date` and `time`.
   - **Otherwise**: create a new log now and remember its path for the rest of the conversation.
   - Updating = read the file, merge new content into existing sections, never truncate it, never touch any OTHER session file.

## Step 2: One question

Single `AskUserQuestion` call, two fields:
- **Topic** — suggest 2-3 lowercase hyphenated names from the conversation (e.g. `auth-middleware-refactor`, `mtce-681-csp-nonce`). If an existing log in `Sessions/` covers the same work (same ticket ID or topic), list it first as `continue: {filename}` so the user can append a follow-up instead of starting a new log. User picks or types own. (On update within the same conversation: reuse the existing topic, don't re-ask.)
- **Notes** — "Any extra notes? (skip to skip)".

## Step 3: Write the log

Auto-select which sections to include based on what actually happened in the session — omit empty ones. Quick Reference, Quick Resume Context, and Raw Session Log are always included.

Tags: lowercase, hyphenated, at most 8. Cover services/repos, frameworks/libraries, action type (refactor/fix/setup/debug/implement/test), tools, ticket IDs. Reuse spellings already used in this project — check first:
```bash
grep -h '^tags:' {vault}/Projects/{folder}/Sessions/*.md | tr -d '[]' | tr ',' '\n' | sed 's/^ *//' | sort | uniq -c | sort -rn | head -40
```
(`symfony` not `Symfony`, `api-platform` not `API-Platform`.) The `**Topics:**` line in Quick Reference repeats the same tags.

Model: lowercase `{family} {major.minor}`, add ` (1m)` for a 1M-context variant, several models comma-separated — e.g. `fable 5.1`, `opus 4.8 (1m)`, `sonnet 5, opus 5`. You are told the model in system context.

Status: `completed`, or `in-progress` when the work is unfinished (compress run mid-task, or the session ended with the task open). `/resume` flags an in-progress last session.

Outcome: one sentence, identical in frontmatter `outcome:` and in Quick Reference. The frontmatter copy is what the vault dashboard (Dataview) displays; the body copy is for humans reading the note.

Filename: `{YYYY-MM-DD}-{HH_MM}-{topic}.md` → `{vault}/Projects/{folder}/Sessions/`.

```markdown
---
type: session
date: {YYYY-MM-DD}
time: "{HH:MM}"
updated: {YYYY-MM-DD}
project: {project-name}
topic: {topic-name}
model: "{model-name}"
tags: [{tags}]
status: completed
outcome: "{one-sentence result}"
---

# Session: {DD-MM-YYYY} {HH:MM} — {Topic Name}

## Quick Reference
**Topics:** {comma-separated tags}
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

## Follow-up — {YYYY-MM-DD}
{only when appending to an existing log from a new conversation: what changed since the last save — decisions, fixes, files, errors. One section per follow-up day.}

## Quick Resume Context
{2-3 sentences optimized for /resume to understand this session fast}

---

## Raw Session Log
{Thorough summary of the full conversation — key code, decisions, errors, solutions.
A summary, not a literal copy. NEVER read by /resume; kept for full-text search.}
```

## Step 4: Confirm

```
Session saved: Projects/{folder}/Sessions/{filename}   (or: Follow-up appended to: {filename})
Sections: {list}
Tags: {tags}
Resume: /resume {topic}
```

## Rules

- **Privacy & scope**: only ever write inside the resolved vault path above. Never read, list, or reference any other Obsidian vault. Never record personal directory paths, usernames, e-mails, API tokens — substitute `<vault>`/`<user>` or omit.
- Frontmatter must be valid YAML — quote strings containing colons, tags as an array.
- Filename date: `YYYY-MM-DD` (sorting). Heading date: `DD-MM-YYYY` (readability).
- On update or follow-up: keep the original filename, `date` and `time`; set `updated` to today; merge, don't overwrite.
- `status` is `completed` or `in-progress` — nothing else. Refresh `status` and `outcome` on every save so they describe the current state.
