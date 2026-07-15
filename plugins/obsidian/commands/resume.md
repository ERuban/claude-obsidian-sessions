---
description: Load context from Obsidian vault + recent session logs for current project
model: sonnet
allowed-tools: Read, Glob, Bash, Grep
---

# Resume Session Context

You are loading context for a new working session. Be thorough but efficient — read summaries, not raw logs.

## Step 0: Resolve the vault path

Two config values control this: `${user_config.vault_path}` (optional override) and `${user_config.vault_name}`.

1. **If `${user_config.vault_path}` is set (non-empty)**, that is the vault base — call it `{vault}` (expand a leading `~`). Obsidian's registry is NOT consulted. This command is read-only: if the folder doesn't exist, stop and tell the user (do NOT create anything).
2. **Otherwise** resolve it from Obsidian's vault registry by matching the vault name `${user_config.vault_name}`:
   1. Read Obsidian's config file (pick the one for the current OS):
      - macOS: `~/Library/Application Support/obsidian/obsidian.json`
      - Linux: `~/.config/obsidian/obsidian.json`
      - Windows: `%APPDATA%\obsidian\obsidian.json`
   2. It contains a `vaults` map: `{ "<id>": { "path": "/abs/path/to/vault", ... }, ... }`. Find the entry whose folder name (last path segment) equals `${user_config.vault_name}`. That `path` is the vault base — call it `{vault}`.
   3. If no vault matches, stop and tell the user the vault name wasn't found in Obsidian's registry, and list the vault folder names that ARE present so they can fix the config (or set the `vault_path` override).

## Step 1: Parse Arguments

The user may pass arguments: `$ARGUMENTS`

Parse them:
- A number means "load last N sessions" (default: 3, max: 50)
- A word/phrase means "search for this topic across all sessions"
- Both can be combined: `5 auth` = last 5 sessions + search for "auth"
- No arguments = last 3 sessions, no topic search

## Step 2: Detect Project

Determine which project you're working in:

1. Check current working directory (`pwd`)
2. Project name = last path segment of the current directory
   - Example: working in `/somewhere/my-project` → project = `my-project`
3. Map to vault project folder. The vault base is `{vault}` (resolved in Step 0)
   - Project folder pattern: `Projects/{project-short-name}/`
   - Use Glob to find the matching project folder: `Projects/*/`
4. If no matching project folder exists, note this — you'll suggest creating one at the end

## Step 3: Read Project Memory

Look for the project's `CLAUDE.md` file in these locations (in order):
1. `{vault}/Projects/{project-folder}/CLAUDE.md`
2. `{project-root}/CLAUDE.md`
3. `{project-root}/.claude/CLAUDE.md`

Read the first one found. This is the persistent memory — architecture decisions, conventions, key paths, status.

## Step 4: Read Session Logs

Session logs are stored at: `{vault}/Projects/{project-folder}/Sessions/*.md`

1. List all session log files, sorted by name (newest first — filenames start with date)
2. For each of the last N sessions:
   - Read everything BEFORE `## Raw Session Log` marker (NEVER read past this marker — it's huge and wastes tokens)
   - Extract: date, topic, Quick Reference (keywords, projects, outcome), decisions, pending tasks
3. If fewer logs exist than requested, note the actual count

## Step 4b: Read Active Kanban Tasks (optional)

If the project keeps tracked tasks as individual files, read them too:
- `{vault}/Projects/{project-folder}/Ready-to-Dev/*.md` — backlog
- `{vault}/Projects/{project-folder}/InWork/*.md` — currently in progress

For each task file (frontmatter `type: task`), extract: title (H1), priority, severity, ticket, source-session, one-line summary from the `## Why` section. Don't read the full body — just the metadata + first paragraph.

If either folder doesn't exist, skip it silently.

## Step 5: Topic Search (if keyword provided)

If the user provided a search topic:
1. Use Grep to search across all session logs in the project folder for the keyword
2. Also search in `## Quick Reference` sections for matching keywords
3. Return up to 5 most relevant matches (by recency)
4. Read their summaries (again, stop before `## Raw Session Log`)

## Step 6: Output Combined Context Report

Format the output as:

```
══════════════════════════════════════
 SESSION RESUMED: {project-name}
══════════════════════════════════════

## Project Status
{From CLAUDE.md — current phase, active work, blockers}

## Most Recent Session
**{date} — {topic}**
{Summary, decisions, outcome}

## Previous Sessions
{List with date, topic, outcome — one line each}

## Pending Tasks
**In Work:** {tasks from InWork/ if present}
**Ready to Dev:** {tasks from Ready-to-Dev/ if present}
**From recent sessions:** {unchecked checkboxes from session logs}

{If topic search was performed:}
## Related Sessions: "{keyword}"
{Matching sessions with date, topic, relevant excerpt}

## Ready to Work
{2-3 sentence summary of where things stand and what's next}
══════════════════════════════════════
```

## Step 7: Handle Edge Cases

- **No project folder in vault**: Suggest creating one. Show the command.
- **No CLAUDE.md**: Mention it.
- **No session logs**: Skip that section, note "No previous sessions logged."
- **Project not detected**: Ask the user which project they're working on.

## Important Rules

- **Privacy & scope**: only read inside the resolved work vault path. Never search, list, or reference any other Obsidian vault. Never echo personal directory paths, usernames, e-mails, or API tokens back to the user.
- NEVER read `## Raw Session Log` sections — they contain full conversations and will blow up context.
- Keep the output concise but complete — this is a briefing, not a novel.
- Always end with actionable next steps.
