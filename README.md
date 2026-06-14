# Projects to Kanban

A Claude Skill that keeps an Airtable kanban board in sync with a local folder of project subfolders. Drop it in to give every project folder on your computer a matching card on a board — name, category, status, and an honest note on what's actually in there.

## What it does

Each immediate subfolder of your projects folder becomes one record on an Airtable board:

- **Name** — the folder name
- **Category** — a short label for the kind of project (e.g. "AI", "Electronics", "Web App")
- **Notes** — a 1-3 sentence honest summary based on what's actually in the folder (README, file names, config files — never guessed from the folder name alone)
- **Status** — `idea`, `inprogress`, `onhold`, `done`, or `archived`

It has two modes:

1. **Setup** (first run) — creates an Airtable base and table with the right fields, and saves a small config file in your projects folder pointing at them.
2. **Sync** (every run after that) — scans your projects folder, compares it against the Airtable board, and creates/updates/flags records to match what's actually there.

## Why "honest"?

A naive version of this just guesses what a project is from its folder name — which goes badly the moment a folder's name doesn't match what's inside it. This skill is built around one rule: anything written to Category or Notes has to be traceable to something it actually read (a README, a file, a folder structure). If a folder is empty or unclear, the skill says so plainly instead of inventing a plausible-sounding description.

## Using this skill

This is a [Claude Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) — a packaged set of instructions Claude can follow when it recognizes a relevant request.

### Install

1. Download or clone this repo.
2. Add the `projects-to-kanban` folder (containing `SKILL.md` and `scripts/`) to your Claude skills directory, or upload it as a `.skill` package if your Claude environment supports that.

### Trigger it

Once installed, just talk to Claude naturally:

- **First time** — "set up a project tracker for my projects folder", "create a kanban board for my projects", or "connect my projects folder to Airtable"
- **Every time after** — "update my project board", "sync my projects to Airtable", "refresh the kanban", or "what's changed across my projects folder?"

Claude will connect to your projects folder, walk through setup or sync automatically, and report back what changed in plain language.

### Requirements

- An Airtable account (the skill creates the base and table for you on first run)
- Claude access to your projects folder
- Claude access to Airtable (via an MCP connector)

## Files

- `SKILL.md` — the skill's instructions and triggering description
- `scripts/setup_airtable.py` — helper used during first-time setup to provision the Airtable base, table, and fields

## License

Use, modify, and share freely.
