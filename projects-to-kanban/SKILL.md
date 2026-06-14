---
name: projects-to-kanban
description: Sync a local projects folder into an Airtable kanban board — scans each project, reads its files to write an honest status summary, and creates/updates/flags Airtable records to match. Use for "update my project board", "sync my projects to Airtable", "refresh the kanban", keeping a tracker in sync with files, or checking what's changed across projects. Also use for first-time setup — "set up a project tracker", "create a kanban board for my projects", "connect my projects folder to Airtable".
---

# Projects to Kanban

Keep an Airtable kanban board in sync with a local folder of project subfolders — one record per project folder, with a short honest description, a category, and a status.

This skill has two modes:

1. **Setup** (first run, or whenever no config file exists): provision the Airtable base/table/fields and write a config file.
2. **Sync** (every other run): scan the projects folder, compare against Airtable, and create/update/flag records.

## Why this skill exists

A naive version of this task invites the model to *guess* what a project is about purely from its folder name — a name that sounds like one thing might actually be something completely different once you look inside. Guesses like this end up in the user's kanban board looking like facts, and the user has no way to tell a real summary from a hallucinated one without opening every folder themselves.

The fix is simple but non-negotiable: **every Category and Notes value written to Airtable must be traceable to something actually read from that project's folder** (a README, a config file, file/folder names, code comments — real artifacts). If a folder is empty, has no readable content, or its purpose genuinely isn't clear from what's there, say so plainly in Notes ("folder is empty" / "couldn't determine purpose from contents — needs a README") rather than inventing a plausible-sounding description. A short honest note beats a confident wrong one.

## Mode 1: Setup

Run setup if `projects-to-kanban.config.json` doesn't exist in the projects folder root, or if the user explicitly asks to set up / reconnect / re-point the board.

1. **Get the projects folder.** If the user hasn't already shared a folder, ask them to connect the parent folder containing their project subfolders (via the cowork directory request tool). Each immediate subfolder = one project.

2. **Get the user's email** (for naming/labeling the Airtable base — e.g. "Projects — {their email}"). It's available in context as the user's email; confirm with the user if it's ambiguous which Airtable account they mean.

3. **Run `scripts/setup_airtable.py`** (or do the equivalent steps directly via the Airtable MCP tools if the script can't run in this environment — see the script for the exact sequence). It will:
   - Search existing bases for one already named for this user/purpose; offer to reuse it if found.
   - Otherwise create a new base ("Projects — {email}") with a "Projects" table.
   - Ensure the table has these fields, creating any that are missing:
     - **Name** (primary/title field, single line text)
     - **Notes** (long text) — the honest summary
     - **Category** (single line text)
     - **Status** (single select, with choices: `idea`, `inprogress`, `onhold`, `done`, `archived`)
     - **Due** (date, optional)
   - Write `projects-to-kanban.config.json` to the root of the projects folder with the resolved base ID, table ID, and field IDs.

4. Tell the user setup is done and that you're now starting the sync.

Re-run setup any time field IDs in the config stop resolving (e.g. user manually renamed/deleted a field in Airtable) — treat a 422/"field does not exist" error as a signal to re-check the config against `list_tables_for_base` / `get_table_schema` rather than assuming the config is still correct.

## Mode 2: Sync

### Step 1 — Inventory both sides

- List every immediate subfolder of the projects folder. Each one is a "project."
- Pull all current records from the Airtable table (`list_records_for_table`), reading Name, Category, Notes, Status.
- Match folders to records by name. Names should match exactly or near-exactly (the record was likely created from the folder name originally) — use judgment for small differences (e.g. capitalization, hyphens vs spaces), but if a match is ambiguous, treat the folder as new rather than guessing.

This gives you three buckets: **matched** (folder + record both exist), **new folders** (no record yet), **orphaned records** (record exists, folder no longer found).

### Step 2 — Read each project folder for real information

For every folder in the "matched" and "new folders" buckets, gather real evidence before writing anything:

1. Look for a README (`README.md`, `README.txt`, `readme`, etc.) at the top level — if present, this is the primary source. Read it.
2. If no README, or it's thin, look at: top-level file names and extensions, subfolder names, any `package.json` / `requirements.txt` / project config files (these often have a `description` field), and any obviously-named key files (e.g. `index.html`, main scripts, design files).
3. Skim — don't deep-read every file in large folders. The goal is "what is this project, roughly, and what state is it in," not a full audit. A few minutes of looking at top-level structure plus the README is usually enough.
4. **Never open files that look like they contain credentials, API keys, tokens, or passwords** (e.g. `.env`, files named `*secret*`, `*credentials*`, `*password*`). If a README or notes file mentions login details, you may note that "login/setup notes are saved here" without reproducing the actual credentials anywhere — not in Notes, not in chat.

From this evidence, draft for each project:

- **Category**: a short label for what kind of project it is (e.g. "AI", "Electronics", "Affiliate Marketing", "App", "CNC", "Web App"). Multiple categories are fine if genuinely applicable ("AI / Affiliate Marketing"). Keep it to a few words.
- **Notes**: 1-3 sentences, plain language, describing what the project actually is and what's in the folder, based only on what you read. If the folder is empty or unclear, say that instead of guessing.

### Step 3 — Decide on Status (ask the user first)

Before touching Status on any existing record, ask the user (once, for the whole sync run) which of these they want:

- **Leave Status alone** — don't touch the Status field on existing records at all. (Good default if they maintain status manually.)
- **Auto-update from evidence** — infer Status from what you found (e.g. a folder with active recent-looking work and no "done"/"shipped" signals → `inprogress`; a folder that's just a name with nothing in it, or explicitly described as an idea/plan → `idea`; a README or file that says something is finished/shipped/launched → `done`). When auto-updating, only change Status if the evidence reasonably supports a different value than what's there — don't churn a record between `inprogress` and `done` based on weak signals, and never set `archived` automatically.

New project records always get a Status — if the user chose "leave alone," still set a sensible initial Status for *new* records only (default `idea` unless the folder clearly has active work, in which case `inprogress`), since a brand-new record can't be "left alone."

### Step 4 — Apply changes

- **Matched folders**: update Notes and Category (always, since these reflect current real content). Update Status only per the user's choice from Step 3.
- **New folders**: create a record with Name = folder name, Category, Notes, and Status as above.
- **Orphaned records** (folder gone): do **not** delete. Instead, prepend `[FOLDER MISSING] ` to the Notes field (if not already there) so the user can see it on the board and decide what to do. Leave Category and Status as-is.

Batch the Airtable writes (`create_records_for_table` / `update_records_for_table` support multiple records per call) rather than one call per record.

### Step 5 — Report back

Summarize for the user in plain terms:

- How many records were updated, created, and flagged as missing.
- For each, a one-line gist of what changed (e.g. "CNC Projects → Category: CNC, marked in-progress").
- Anything you skipped or couldn't determine, and why (e.g. "Folder X is empty, marked as such rather than guessing").
- If Status auto-update was on, call out any Status changes specifically since those are the most visible on a kanban board.

Keep the report conversational and skimmable — this person doesn't want a wall of bullet points, just a quick honest rundown of what moved.

## Files in this skill

- `scripts/setup_airtable.py` — provisions the Airtable base/table/fields and writes the config file. Read this script before running setup so you understand exactly what it does and can adapt the steps to direct MCP tool calls if the script itself can't execute in your environment.
                                                                                                                                                           