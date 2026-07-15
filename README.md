# Obsidian Tools — Claude Code Plugin & Marketplace

A Claude Code plugin that tracks your work sessions in an **Obsidian vault**. This repo is **both** a plugin marketplace (`obsidian-tools`) **and** the plugin itself (`obsidian`).

Two slash commands, invoked namespaced after install:

- **`/obsidian:compress`** — save the current Claude session as a structured, searchable session log in your vault.
- **`/obsidian:resume`** — load project context (CLAUDE.md, recent session logs, kanban tasks) for the project you're working in.

## How it works

- You normally configure **one value**: the **vault name**. The plugin resolves the vault's full path automatically from Obsidian's own vault registry (`obsidian.json`), so nothing is hardcoded.
- Optionally set **`vault_path`** (absolute path to the vault folder) to bypass the registry — for machines without Obsidian installed (headless servers, devcontainers) or to disambiguate same-named vaults. When set, it takes precedence over the registry lookup.
- **Project detection** is automatic: the commands use the name of your current working directory (`pwd` basename) as the project name, and map it to `Projects/{project}/` inside the vault.
- **First `/compress` of a project scaffolds it**: `Projects/{project}/Sessions/` + `Ready-to-Dev/` are created automatically, and missing vault-level files (`Templates/*.md`, `Home.md` dashboard) are seeded from the plugin's bundled templates — create-only, existing files are never touched. No separate init step.

## Non-destructive guarantee

Enabling the plugin does **nothing** to your vault — it only stores the vault name (and the optional path). No files are created, moved, or deleted at install time.

- `/obsidian:compress` scaffolds strictly create-only: a new project gets `Sessions/` + `Ready-to-Dev/`, missing vault-level files (`Templates/*.md`, `Home.md`) are seeded from the bundled templates, then it creates or merges into a single session log file for the current Claude session. It never overwrites or touches any existing note. With `vault_path` set, it will also create the vault folder itself if it doesn't exist yet — note that Obsidian won't discover such a fresh folder on its own: open it once via "Open folder as vault" (the `Home.md` dashboard additionally needs the community Dataview plugin).
- `/obsidian:resume` is **read-only** (with `vault_path` set it reports a missing folder instead of creating it).

Point the config at your **existing** vault — your notes are safe.

## Install

In Claude Code:

```
/plugin marketplace add ERuban/claude-obsidian-sessions
/plugin install obsidian@obsidian-tools
/reload-plugins
```

Or from the terminal, non-interactive (sets the vault name in one go; `--config` also works on an already-installed plugin to set or change the value):

```
claude plugin marketplace add ERuban/claude-obsidian-sessions
claude plugin install obsidian@obsidian-tools --config vault_name=<your-vault-name>
```

Add `--config vault_path=/abs/path/to/vault` to skip the Obsidian registry lookup (see "How it works").

Note: there is no `claude plugin config` subcommand — use `install --config` from the terminal, or `/plugin configure obsidian@obsidian-tools` inside Claude Code.

For local testing before pushing:

```
/plugin marketplace add /absolute/path/to/claude-obsidian-sessions
```

On enable, enter the `userConfig` values (or pass them via `--config` as above):

- **Vault name** (required) → the name of your Obsidian vault (the vault folder name as shown in Obsidian, e.g. `my-vault`).
- **Vault path** (optional) → absolute path to the vault folder; overrides the registry lookup when set.

The plugin looks this name up in Obsidian's vault registry and resolves the full path itself — you don't type any path.

## Verify

- `/plugin list` shows `obsidian`.
- `/obsidian:compress` and `/obsidian:resume` are available.
- Run `/obsidian:resume` first (read-only) to confirm it reads existing sessions without altering them.

## Layout

```
claude-obsidian-sessions/
├── .claude-plugin/
│   └── marketplace.json          # marketplace: obsidian-tools
├── plugins/
│   └── obsidian/
│       ├── .claude-plugin/
│       │   └── plugin.json        # plugin: obsidian + userConfig (vault_name)
│       ├── commands/
│       │   ├── compress.md        # /obsidian:compress
│       │   └── resume.md          # /obsidian:resume
│       └── templates/             # seeded into the vault on first /compress (create-only)
│           ├── Home.md
│           ├── Project-CLAUDE.md
│           ├── Session.md
│           └── Task.md
└── README.md
```
