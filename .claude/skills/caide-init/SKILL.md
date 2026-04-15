---
name: caide-init
description: Interactive bootstrap wizard that generates a CLAUDE.md, initializes the wiki/ scaffold, installs CAIDE skills, and configures .obsidian/app.json for the current repository. Run once at project setup. Trigger: /caide-init
trigger: /caide-init
---

# /caide-init

## Purpose
Bootstrap a new CAIDE project in the current repository. This skill walks the user
through every setup step interactively, then writes all required files automatically.
Run it once at the start of a project. It is safe to re-run on an existing project —
all existing files are preserved; only missing pieces are created.

## Trigger
`/caide-init`

---

## STRICT OPERATING RULES

These rules override all default assistant behavior and MUST be followed at all times:

1. **No independent file evaluation.** Do NOT read, enumerate, glob, or inspect ANY
   source files, directories, or file extensions at any point in this skill UNLESS
   the user explicitly chooses AUTO-DISCOVER for that specific phase. Even then,
   perform only the exact operations described in that phase — nothing more.

2. **No user-level settings checks.** Only check paths relative to the current working
   directory (the repo root). Never access `~/.claude/`, user-global paths, or any path
   outside the repo.

3. **No phase proceeds without user confirmation.** After each phase completes, explicitly
   wait for the user to confirm they are ready to continue. Do not advance automatically.

4. **Preflight is the only phase that runs without a preceding prompt.** All subsequent
   phases require the user to respond before any action is taken.

5. **Ask, then act.** For every phase, present the questions first, wait for the user's
   full response, then perform any file operations. Never interleave questions with
   file reads or writes.

6. **Do exactly what the phases describe — nothing else.** Do not attempt to understand
   the codebase, summarize code patterns, or make inferences beyond what is explicitly
   instructed in each phase.

---

## Workflow

### STEP 0 — Display Phase Checklist

Before doing anything else, display this task list to the user:

```
CAIDE Bootstrap — Phase Plan
─────────────────────────────────────────────────────
[ ] Phase 0 — Preflight Check      (runs now, no input needed)
[ ] Phase 1 — Project Identity     (requires your input)
[ ] Phase 2 — Source Directories   (requires your input)
[ ] Phase 3 — Skills Installation  (requires your input)
[ ] Phase 4 — Generate CLAUDE.md   (writes files after your confirmation)
[ ] Phase 5 — Wiki Scaffold        (writes files after your confirmation)
[ ] Phase 6 — Obsidian Config      (writes files after your confirmation)
[ ] Phase 7 — Final Report         (summary only)
─────────────────────────────────────────────────────
Starting Phase 0 — Preflight Check…
```

Then immediately proceed to Phase 0. No user input is needed before Phase 0.

---

### PHASE 0 — Preflight Check

Check ONLY the following explicit file and directory paths — relative to the repo root.
Do NOT walk the directory tree, enumerate files, or check any other paths.

**Exact paths to check:**

| Path                       | What to note                                              |
|----------------------------|-----------------------------------------------------------|
| `./CLAUDE.md`              | Exists → will ask user how to handle it                   |
| `./wiki/`                  | Exists → scaffold files will not be overwritten           |
| `./wiki/index.md`          | Exists → will skip creation                               |
| `./wiki/log.md`            | Exists → will skip creation                               |
| `./wiki/overview.md`       | Exists → will skip creation                               |
| `./wiki/hot.md`            | Exists → will skip creation                               |
| `./.claude/skills/scan/`   | Exists → skill already installed                          |
| `./.claude/skills/health/` | Exists → skill already installed                          |
| `./.claude/skills/impact/` | Exists → skill already installed                          |
| `./.claude/skills/adr/`    | Exists → skill already installed                          |
| `./.claude/skills/recall/` | Exists → skill already installed                          |
| `./.claude/skills/trace/`  | Exists → skill already installed                          |
| `./.obsidian/app.json`     | Exists → will merge rather than overwrite                 |

After checking all paths, report the preflight summary as a bulleted list:

```
Preflight Results:
  • CLAUDE.md         — [found | not found]
  • wiki/             — [found | not found]
  • wiki/index.md     — [found | not found]
  • wiki/log.md       — [found | not found]
  • wiki/overview.md  — [found | not found]
  • wiki/hot.md       — [found | not found]
  • skills/scan       — [installed | missing]
  • skills/health     — [installed | missing]
  • skills/impact     — [installed | missing]
  • skills/adr        — [installed | missing]
  • skills/recall     — [installed | missing]
  • skills/trace      — [installed | missing]
  • .obsidian/app.json — [found | not found]
```

**Decision logic:**

- If ALL files are present AND all skills are installed → inform the user that CAIDE
  appears to be fully configured, display the checklist as all checked, and ask:
  > "CAIDE is already set up in this repository. Would you like to (a) re-run the
  > full bootstrap to update or fix anything, or (b) exit?"
  If the user chooses to exit, stop here. Do not proceed further.

- If ANY file is missing OR any skill is missing → continue to Phase 1.
  Before continuing, tell the user exactly which items are missing and ask:
  > "Some items are missing. I will now walk you through each phase to set them up.
  > Ready to continue? (yes/no)"
  Wait for the user to respond "yes" before proceeding.

Mark Phase 0 as complete in the checklist: `[x] Phase 0 — Preflight Check`

---

### PHASE 1 — Project Identity

**Show the updated checklist** with Phase 0 checked, Phase 1 in progress:
```
[x] Phase 0 — Preflight Check
[→] Phase 1 — Project Identity
[ ] Phase 2 — Source Directories
...
```

Ask the user ALL of the following questions in a single numbered list.
Wait for the user's complete response before doing anything else.

```
Phase 1 — Project Identity

1. What is the project name?

2. How would you like to identify the primary programming language(s), build system,
   and domain?

   (a) AUTO-DISCOVER — I will search for build manifest files only (e.g.,
       CMakeLists.txt, package.json, Cargo.toml, etc.) and inspect file extension
       distribution at the root and one level deep. I will NOT read any file contents.
       I will show you my findings before writing anything.

   (b) MANUAL — You describe them yourself.
       Example: "C++17, CMake — real-time graphics engine for AAA games"
```

Wait for the user's response. Then:

**If the user chose (a) AUTO-DISCOVER:**
- Glob ONLY for these exact build manifest filenames at the repo root:
  `CMakeLists.txt`, `package.json`, `Cargo.toml`, `pyproject.toml`, `Makefile`,
  `go.mod`, `mix.exs`, `build.gradle`, `*.csproj`, `*.sln`, `*.xcodeproj`, `*.cabal`
- Enumerate file extensions found in the root and one level deep. Do NOT read any
  file contents. Do NOT traverse deeper than two levels.
- Summarize findings to the user. Example:
  > "I detected: build files → CMakeLists.txt; common extensions → .cpp, .h, .glsl"
- Ask the user to confirm or correct before continuing.

**If the user chose (b) MANUAL:**
- Accept the user's description as-is.

Store the confirmed result as `{PROJECT_DESCRIPTION}` — a single line suitable for the
`## Project` section of CLAUDE.md.

After the user has confirmed their answers, tell them Phase 1 is complete and ask:
> "Phase 1 complete. Ready to move to Phase 2 — Source Directories? (yes/no)"

Wait for confirmation before proceeding.

Mark Phase 1 complete: `[x] Phase 1 — Project Identity`

---

### PHASE 2 — Source-Like Directories

**Show the updated checklist.**

Ask the user ALL of the following in a single message.
Wait for the user's complete response before taking any action.

```
Phase 2 — Source-Like Directories

3. How would you like to identify your source-like directories?

   (a) AUTO-DISCOVER — I will walk the directory tree ONE level deep and classify
       directories by their contents. I will NOT read any file contents. I will
       use any .gitignore / .claudeignore files found at the repo root to exclude
       noise. I will show you my findings before writing anything.

   (b) MANUAL — List the directories yourself.
       Example: src/, lib/, assets/shaders/
```

Wait for the user's response. Then:

**If the user chose (a) AUTO-DISCOVER:**

a. Read ONLY these specific ignore files if they exist at the repo root — do not
   search for them recursively:
   `.gitignore`, `.claudeignore`, `.p4ignore`, `.npmignore`, `.hgignore`
   Extract directory-level exclusion patterns (lines ending in `/` or bare directory
   names). Always exclude these regardless of ignore files:
   `node_modules/`, `.git/`, `.svn/`, `.hg/`, `dist/`, `build/`, `out/`,
   `target/`, `__pycache__/`, `.cache/`, `coverage/`, `.next/`, `.nuxt/`,
   `vendor/`, `Pods/`, `.gradle/`, `bin/`, `obj/`, `tmp/`, `temp/`,
   `wiki/`, `.claude/`, `.obsidian/`, `.idea/`, `.vscode/`

b. Walk ONLY one level deep from the repo root. For each non-excluded directory:
   - Classify as **source-like** if it contains source file extensions.
   - Classify as **asset-like** if it contains only non-code files.
   - Classify as **config-like** if it contains only config files.
   - Classify as **script-like** if it contains script files.

c. Present findings in a table:

   ```
   Detected directories:
   ┌──────────────────────┬────────────┬──────────────────────────┐
   │ Directory            │ Type       │ Primary extensions found  │
   ├──────────────────────┼────────────┼──────────────────────────┤
   │ src/                 │ source     │ .cpp, .h                  │
   └──────────────────────┴────────────┴──────────────────────────┘

   Which of these should be included in ## Source-Like Directories?
   (Enter letters, "all", or tell me what to add/remove.)
   ```

d. Wait for the user's selection.

**If the user chose (b) MANUAL:**
Ask the user to list directories and provide a one-line description for each.

Store the confirmed list as `{SOURCE_DIRS}`.

After the user has confirmed, tell them Phase 2 is complete and ask:
> "Phase 2 complete. Ready to move to Phase 3 — Skills Installation? (yes/no)"

Wait for confirmation before proceeding.

Mark Phase 2 complete: `[x] Phase 2 — Source Directories`

---

### PHASE 3 — Skills Installation

**Show the updated checklist.**

Using the preflight results from Phase 0, present the skills status to the user.
Check ONLY `.claude/skills/` relative to the current repo root — do NOT check
user-global paths, `~/.claude/`, or any path outside this repository.

```
Phase 3 — Skills Installation

CAIDE Skills Status (repo-local only: .claude/skills/):
  /scan    — [installed | missing]
  /health  — [installed | missing]
  /impact  — [installed | missing]
  /adr     — [installed | missing]
  /recall  — [installed | missing]
  /trace   — [installed | missing]

Skills are installed repo-locally under .claude/skills/.
Would you like me to install any missing skills? (yes/no)
```

Wait for the user's response.

**If the user says yes:**

Ask for the CAIDE repository URL before attempting any downloads:
> "What is the CAIDE skills repository URL?
> (Leave blank to skip skill installation and do it manually later.)"

Wait for the user's response. If they provide a URL, use it. If they leave it blank,
skip installation and note it in the final report.

If a URL is provided, for each missing skill (e.g., `scan`), fetch:
```
{SKILLS_REPO}/raw/main/.claude/skills/scan/SKILL.md → .claude/skills/scan/SKILL.md
```

Run the download using `curl -fsSL` or `wget -q -O`. If neither is available,
instruct the user to manually copy the files and provide the URL.

After attempting downloads, report success/failure for each skill.

**If the user says no:**
Note all skills as skipped in the final report.

After the user's response is handled, ask:
> "Phase 3 complete. Ready to move to Phase 4 — Generate CLAUDE.md? (yes/no)"

Wait for confirmation before proceeding.

Mark Phase 3 complete: `[x] Phase 3 — Skills Installation`

---

### PHASE 4 — Generate CLAUDE.md

**Show the updated checklist.**

Before writing anything, confirm with the user:

> "I am ready to write CLAUDE.md. Here is a summary of what will be generated:
>
> - Project: {PROJECT_DESCRIPTION}
> - Source dirs: {SOURCE_DIRS}
>
> Shall I write CLAUDE.md now? (yes/no)"

Wait for the user's confirmation. Only write the file after receiving "yes".

If the user chose to UPDATE an existing CLAUDE.md (not reinitialize), only write
sections that are absent from the existing file. Do not touch sections already present.

Using the collected values, write `CLAUDE.md` at the repo root using the template below,
substituting `{PROJECT_DESCRIPTION}` and `{SOURCE_DIRS}` with the values collected
in Phases 1 and 2.

````markdown
# CAIDE — Master Schema

## Project
{PROJECT_DESCRIPTION}

## Project Structure
- `src/` — source code. LLM may modify with explicit user permission.
- `wiki/` — LLM-generated intelligence layer. LLM owns this entirely.
- `wiki/index.md` — master catalog. Update on EVERY scan.
- `wiki/log.md` — append-only operation log. Never delete entries.
- `wiki/overview.md` — high-level synthesis. Revise after major scans.
- `wiki/hot.md` — session hot cache (~500 words). Read silently at session
  start BEFORE responding.
- `CLAUDE.md` — this file. Re-read at the start of every session.

## Source-Like Directories
The following directories outside `src/` are also indexed:
{SOURCE_DIRS_FORMATTED}
[ADD OR REMOVE entries to match your project structure]

## Page Conventions
Every wiki page MUST have YAML frontmatter. Use these schemas:

### Module/File Pages (wiki/modules/)
---
type: module
title: "Module Name"
src_path: src/path/to/File.ext
language: {detected_primary_language}
exports:
  - SymbolName::method()
imports:
  - [[wiki-link-to-imported-module]]
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: high | medium | low
---

### Dependency/Link Pages (wiki/dependencies/)
---
type: dependency
title: "Dependency: A → B"
from: [[module-a]]
to: [[module-b]]
relationship: imports | inherits | instantiates | calls | configures
direction: unidirectional | bidirectional
impact: high | medium | low
notes: "Why this dependency exists and what would break if removed"
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

### Component/Class Pages (wiki/components/)
---
type: component
component_type: class | struct | function | interface | shader | config
title: "ComponentName"
defined_in: [[module-name]]
signature: "class ComponentName : public IBase"
role: "One-sentence architectural role description"
dependencies:
  - [[dep-link]]
consumers:
  - [[module-link]]
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: high | medium | low
---

### Architectural Decision Records (wiki/adrs/)
---
type: adr
adr_number: "001"
title: "Decision title"
status: proposed | accepted | superseded | deprecated
decision_date: YYYY-MM-DD
context: "Why this decision was needed"
decision: "What was decided"
consequences:
  - "Impact 1"
  - "Impact 2"
files_to_modify: []
related_modules: []
---

## Scan Workflow
When I say "/scan [path]" or "scan src/":
1. Parse all source files in the specified path.
2. Identify new, changed, or deleted modules and dependencies.
3. Create or update wiki/modules/ pages for each file.
4. Create or update wiki/dependencies/ pages for each import relationship.
5. Create or update wiki/components/ pages for significant classes/functions.
6. Update wiki/index.md — add/update entries with one-line summaries.
7. Log the operation in wiki/log.md with the structured format below.
A single scan should create or update 5–20 wiki pages depending on scope.

## Query Workflow
When I say "/query" or ask an architectural question:
1. Read wiki/index.md to identify relevant modules and dependencies.
2. Read those pages directly.
3. Synthesize an answer with [[wiki-link]] citations.
4. If the answer reveals a refactor opportunity or decision point, offer to
   file it as a new ADR in wiki/adrs/.
5. Update wiki/log.md with a query entry.

## Health Workflow
When I say "/health":
1. Find orphaned files — files in src/ with no corresponding wiki/modules/ entry.
2. Find broken dependency links — [[wiki-links]] that reference non-existent pages.
3. Identify undocumented changes — src/ files newer than their wiki page updated date.
4. Detect dead code candidates — modules with no inbound dependency links.
5. Detect stale documentation — wiki pages referencing exports absent from src/.
6. Append a health entry to wiki/log.md.

## Impact Workflow
When I say "/impact [symbol or file]":
1. Look up the symbol or file in wiki/modules/ and wiki/components/.
2. Traverse wiki/dependencies/ to find all downstream consumers.
3. Rank by impact: high (direct consumers), medium (transitive), low (indirect).
4. Report a blast-radius summary with affected modules listed by tier.
5. Log the operation in wiki/log.md.

## ADR Workflow
When I say "/adr [decision title or description]":
1. Draft an ADR in wiki/adrs/ using the schema above.
2. Auto-number the ADR (next available adr_number).
3. Populate context from the current conversation or wiki pages.
4. Ask the user to confirm or edit the draft before saving.
5. Log the new ADR in wiki/log.md.

## Recall Workflow
When I say "/recall [topic or date range]":
1. Search wiki/log.md for matching entries.
2. Pull the relevant wiki/modules/, wiki/components/, and wiki/adrs/ pages.
3. Synthesize a narrative of what changed, when, and why.
4. Present in reverse-chronological order with [[wiki-link]] citations.

## Trace Workflow
When I say "/trace [symbol]":
1. Find the symbol's definition in wiki/components/ or wiki/modules/.
2. Walk wiki/dependencies/ upstream to find all callers/consumers.
3. Walk downstream to find all dependencies.
4. Render a dependency chain from origin to leaves.
5. Flag any circular dependencies detected.
6. Log the trace in wiki/log.md.

## Log Format
Each log entry MUST start with this prefix for parsability:
## [YYYY-MM-DD] {scan|query|health|impact|adr|recall|trace} | {description}

## Safety Rules
- NEVER delete wiki pages. Mark as deprecated in frontmatter instead.
- Always ensure wiki/ stays synchronized with src/.
- When modifying src/, proactively identify and suggest necessary updates
  to wiki/ documentation before proceeding.
- Always update wiki/index.md and wiki/log.md on every operation.
- When uncertain about a component's role or relationship, set confidence: low.
- Cross-reference all new module pages to at least 2 existing wiki pages.
````

After writing, confirm to the user what was written. Then ask:
> "Phase 4 complete. Ready to move to Phase 5 — Wiki Scaffold? (yes/no)"

Wait for confirmation before proceeding.

Mark Phase 4 complete: `[x] Phase 4 — Generate CLAUDE.md`

---

### PHASE 5 — Initialize wiki/ Scaffold

**Show the updated checklist.**

Before writing anything, list the files that will be created and ask:
> "I will create the following wiki scaffold files (existing files will not be
> overwritten):
>   • wiki/index.md
>   • wiki/log.md
>   • wiki/overview.md
>   • wiki/hot.md
>
> Shall I proceed? (yes/no)"

Wait for the user's confirmation. Only write files after receiving "yes".

Create the following files if they do not already exist. Never overwrite existing files.

**`wiki/index.md`**
```markdown
---
type: index
updated: {TODAY}
---

# CAIDE Wiki — Index

## Modules
<!-- Populated by /scan -->

## Dependencies
<!-- Populated by /scan -->

## Components
<!-- Populated by /scan -->

## Architectural Decisions
<!-- Populated by /adr -->
```

**`wiki/log.md`**
```markdown
# CAIDE Operation Log

<!-- Append-only. Never delete entries. Most recent first. -->

## [{TODAY}] init | CAIDE bootstrap via /caide-init
Project initialized. CLAUDE.md generated. Wiki scaffold created.
Source-like directories: {SOURCE_DIRS_INLINE}
Skills installed: {SKILLS_INSTALLED_LIST}
```

**`wiki/overview.md`**
```markdown
---
type: overview
updated: {TODAY}
---

# Project Overview

{PROJECT_DESCRIPTION}

## Architecture Summary
<!-- Populated after first /scan -->

## Key Modules
<!-- Populated after first /scan -->

## Key Dependencies
<!-- Populated after first /scan -->
```

**`wiki/hot.md`**
```markdown
---
type: hot-cache
updated: {TODAY}
---

# Session Hot Cache

> Read this silently at the start of every session before responding.

## Project
{PROJECT_DESCRIPTION}

## Last Operation
{TODAY} — init | CAIDE bootstrap

## Watch Points
<!-- High-priority items to keep in mind. Updated by /health and /scan. -->

## Recent ADRs
<!-- Most recent architectural decisions. Updated by /adr. -->
```

After writing, confirm to the user which files were created and which were skipped
(already existed). Then ask:
> "Phase 5 complete. Ready to move to Phase 6 — Obsidian Config? (yes/no)"

Wait for confirmation before proceeding.

Mark Phase 5 complete: `[x] Phase 5 — Wiki Scaffold`

---

### PHASE 6 — Configure .obsidian/app.json

**Show the updated checklist.**

Before writing anything, ask the user:
> "I will configure .obsidian/app.json to exclude source directories from the
> Obsidian vault index, based on the directories you confirmed in Phase 2.
>
> Source dirs to exclude: {SOURCE_DIRS}
>
> Shall I proceed? (yes/no)"

Wait for the user's confirmation. Only write the file after receiving "yes".

**Build the ignore list** from the following sources:
1. All entries in `{SOURCE_DIRS}` collected in Phase 2.
2. Standard non-wiki directories that should always be excluded:
   `src`, `lib`, `assets`, `scripts`, `config`, `build`, `dist`, `out`,
   `target`, `node_modules`, `.git`, `.claude`, `.github`, `vendor`,
   `Pods`, `.gradle`, `coverage`, `tmp`, `temp`, `test`, `tests`,
   `spec`, `specs`, `__pycache__`, `.cache`
3. Any directory discovered in Phase 2 (all non-wiki directories).

**If `.obsidian/app.json` does not exist**, create `.obsidian/` directory and write:

```json
{
  "userIgnoreFilters": [
    "{source_dir_1}",
    "{source_dir_2}",
    "..."
  ]
}
```

**If `.obsidian/app.json` already exists**, read it, then merge the `userIgnoreFilters`
array — adding any new entries without removing existing ones. Write the merged file back.

Format each entry as a plain directory name without trailing slash (Obsidian convention).
For nested paths like `assets/shaders`, add both `assets/shaders` and `assets` as
separate entries so neither level appears in the graph.

After writing, confirm to the user what was written. Then ask:
> "Phase 6 complete. Ready for the Final Report? (yes/no)"

Wait for confirmation before proceeding.

Mark Phase 6 complete: `[x] Phase 6 — Obsidian Config`

---

### PHASE 7 — Final Report

**Show the fully checked checklist:**

```
[x] Phase 0 — Preflight Check
[x] Phase 1 — Project Identity
[x] Phase 2 — Source Directories
[x] Phase 3 — Skills Installation
[x] Phase 4 — Generate CLAUDE.md
[x] Phase 5 — Wiki Scaffold
[x] Phase 6 — Obsidian Config
[x] Phase 7 — Final Report
```

Print a completion summary:

```
╔══════════════════════════════════════════════════════╗
║            CAIDE Bootstrap Complete                  ║
╚══════════════════════════════════════════════════════╝

Project:   {PROJECT_DESCRIPTION}

Files created:
  ✓ CLAUDE.md
  ✓ wiki/index.md
  ✓ wiki/log.md
  ✓ wiki/overview.md
  ✓ wiki/hot.md
  ✓ .obsidian/app.json

Skills installed:
  [list each skill with ✓ installed / ⚠ skipped / ✗ failed]

Source-like directories indexed:
  [list each directory from {SOURCE_DIRS}]

Obsidian ignore filters added: {N} entries

─────────────────────────────────────────────────────
Next steps:
  1. Run /scan {first_source_dir}/ to populate the wiki.
  2. Open the wiki/ folder in Obsidian for the knowledge graph.
  3. Run /health after the first scan to verify everything is connected.
─────────────────────────────────────────────────────
```

---

## Notes

- This skill never reads source file contents at any phase — only filenames,
  extensions, and build manifests (for language/build detection in Phase 1 AUTO only).
  Source content is left entirely to `/scan`.
- Only paths relative to the current working directory (the repo root) are ever checked.
  User-global paths (`~/.claude/`, etc.) are never accessed.
- If a step fails (e.g., skill download fails due to no network), continue with the
  remaining steps and list failures in the final report. Never abort mid-run.
- The `{TODAY}` placeholder should be substituted with the actual current date in
  `YYYY-MM-DD` format wherever it appears in generated file content.
- The `{SOURCE_DIRS_FORMATTED}` placeholder produces a bulleted markdown list:
  ```
  - `src/` — source code (primary)
  - `lib/` — vendored libraries (index exports and public APIs only)
  - `assets/shaders/` — GLSL/HLSL shader files (index uniforms and stage)
  ```
- The `{SOURCE_DIRS_INLINE}` placeholder produces a comma-separated list for the log.
