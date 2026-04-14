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

## Workflow

Work through the phases below sequentially. Complete each phase fully before moving
to the next. Display a phase header to the user as you enter each phase.

---

### PHASE 0 — Preflight Check

Before asking the user anything, silently check the current working directory for:

1. **Existing CLAUDE.md** — if found, inform the user it already exists and ask:
   > "A `CLAUDE.md` already exists. Would you like to (a) reinitialize it from
   > scratch, (b) update only the missing sections, or (c) skip CLAUDE.md generation
   > entirely?"
   Honor the user's answer throughout the rest of this skill.

2. **Existing wiki/ directory** — note whether it exists (you will skip scaffold
   creation for any files already present).

3. **Existing .claude/skills/ directory (local to this repo)** — note which skill
   directories are already present so you know what to install later.

4. **Existing .obsidian/app.json** — if found, note it; you will merge into it
   rather than overwrite it in Phase 5.

Report the preflight summary to the user in a short bulleted list before asking Phase 1
questions.

---

### PHASE 1 — Project Identity

Ask the user the following questions. You may ask them all at once in a numbered list
to minimize back-and-forth.

```
I need a few details to generate your CLAUDE.md.

1. What is the project name?
2. Would you like me to AUTO-DISCOVER the primary programming language(s), build
   system, and domain — or would you prefer to describe them yourself?
   → If AUTO: I will inspect file extensions, build files (CMakeLists.txt,
     package.json, Cargo.toml, pyproject.toml, build.gradle, Makefile, etc.),
     and top-level README to infer these. I will show you my findings before
     writing anything.
   → If MANUAL: Please describe: language(s), build system, and primary domain
     (e.g., "C++17, CMake — real-time graphics engine for AAA games").
```

**If AUTO-DISCOVER chosen:**
- Glob for build manifests: `CMakeLists.txt`, `package.json`, `Cargo.toml`,
  `pyproject.toml`, `*.csproj`, `build.gradle`, `Makefile`, `*.sln`, `*.xcodeproj`,
  `go.mod`, `mix.exs`, `*.cabal`.
- Inspect file extension distribution in the root and up to two levels deep (do NOT
  read file contents — just enumerate extensions).
- Summarize findings to the user: e.g., "I detected: C++ (primary), CMake build
  system, real-time simulation domain (inferred from directory names)."
- Ask the user to confirm or correct before proceeding.

Store the final result as: `{PROJECT_DESCRIPTION}` — a single line suitable for the
`## Project` section of CLAUDE.md.

---

### PHASE 2 — Source-Like Directories

Ask the user:

```
3. How would you like to identify your source-like directories?
   → AUTO: I will walk the directory tree and identify candidate directories.
     I will NOT read file contents. I will use any .gitignore, .claudeignore,
     .p4ignore, or .hgignore files to exclude noise.
   → MANUAL: Tell me which directories to include (e.g., src/, lib/, assets/shaders/).
```

**If AUTO chosen:**

a. Read all ignore files found at the repo root (`.gitignore`, `.claudeignore`,
   `.p4ignore`, `.npmignore`, `.hgignore`). Extract the directory-level exclusion
   patterns (lines ending in `/`, or bare directory names). Common directories to
   always exclude regardless:
   - `node_modules/`, `.git/`, `.svn/`, `.hg/`, `dist/`, `build/`, `out/`,
     `target/`, `__pycache__/`, `.cache/`, `coverage/`, `.next/`, `.nuxt/`,
     `vendor/`, `Pods/`, `.gradle/`, `bin/`, `obj/`, `tmp/`, `temp/`,
     `wiki/`, `.claude/`, `.obsidian/`, `.idea/`, `.vscode/`

b. Walk only one level deep from the repo root. For each directory not excluded:
   - Classify as **source-like** if it contains source file extensions
     (`.cpp`, `.c`, `.h`, `.hpp`, `.ts`, `.tsx`, `.js`, `.jsx`, `.py`, `.rs`,
     `.go`, `.cs`, `.java`, `.kt`, `.swift`, `.rb`, `.ex`, `.elm`, `.hs`,
     `.lua`, `.glsl`, `.hlsl`, `.wgsl`, `.metal`, `.shader`).
   - Classify as **asset-like** if it contains non-code files only
     (`.png`, `.jpg`, `.wav`, `.mp3`, `.ogg`, `.fbx`, `.obj`, `.gltf`).
   - Classify as **config-like** if it contains only `.json`, `.yaml`, `.toml`,
     `.ini`, `.cfg`.
   - Classify as **script-like** if it contains `.sh`, `.ps1`, `.bat`, `.py`
     scripts alongside no primary source.

c. Present findings to the user in a table:

   ```
   Detected source-like directories:
   ┌──────────────────────┬────────────┬──────────────────────────┐
   │ Directory            │ Type       │ Primary extensions found  │
   ├──────────────────────┼────────────┼──────────────────────────┤
   │ src/                 │ source     │ .cpp, .h                  │
   │ lib/                 │ source     │ .cpp, .h                  │
   │ assets/shaders/      │ source     │ .glsl, .hlsl              │
   │ scripts/             │ script     │ .py, .sh                  │
   │ config/              │ config     │ .toml, .yaml              │
   └──────────────────────┴────────────┴──────────────────────────┘

   Which of these should be included in ## Source-Like Directories?
   (Enter directory letters to include, or "all", or tell me what to add/remove.)
   ```

d. Accept the user's selection. Store the confirmed list as `{SOURCE_DIRS}`.

**If MANUAL chosen:**
Ask the user to list directories and provide a one-line description for each.
Store as `{SOURCE_DIRS}`.

---

### PHASE 3 — Skills Installation

The following CAIDE skills power the `/scan`, `/query`, `/health`, `/impact`,
`/adr`, `/recall`, and `/trace` commands.

Check which skill directories already exist under `.claude/skills/` in the **current
repo** (repo-local) and under the **user-global** `~/.claude/skills/` path.

Present status to the user:

```
CAIDE Skills Status:
  /scan    — [installed | missing]
  /health  — [installed | missing]
  /impact  — [installed | missing]
  /adr     — [installed | missing]
  /recall  — [installed | missing]
  /trace   — [installed | missing]

Skills are installed repo-locally under .claude/skills/.
Would you like me to install any missing skills?
```

If the user confirms, clone or download each missing skill from the CAIDE skills
repository:

```
SKILLS_REPO=https://github.com/[CAIDE_REPO_PLACEHOLDER]/claude-llm-code
SKILLS_BASE=$SKILLS_REPO/raw/main/.claude/skills
```

For each missing skill (e.g., `scan`), fetch:
```
$SKILLS_BASE/scan/SKILL.md  →  .claude/skills/scan/SKILL.md
```

Run the download using `curl -fsSL` or `wget -q -O`. If neither is available,
instruct the user to manually copy the files from the repository and provide the URL.

After attempting downloads, report success/failure for each skill.

> **Note:** If the user has not yet provided the CAIDE repository URL (the placeholder
> above), ask them before attempting any downloads:
> "What is the CAIDE skills repository URL? (Leave blank to skip skill installation
> and do it manually later.)"

---

### PHASE 4 — Generate CLAUDE.md

Using the collected values, write `CLAUDE.md` at the repo root. Use the template
below, substituting `{PROJECT_DESCRIPTION}` and `{SOURCE_DIRS}` with the values
collected in Phases 1 and 2.

If the user chose to UPDATE an existing CLAUDE.md (not reinitialize), only write
sections that are absent from the existing file. Do not touch sections already present.

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

---

### PHASE 5 — Initialize wiki/ Scaffold

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

---

### PHASE 6 — Configure .obsidian/app.json

CAIDE uses Obsidian to visualize the wiki knowledge graph. The connections graph
should show only wiki pages (modules, dependencies, components, ADRs) — never
raw source files, which are tracked by `/scan` but not stored in `wiki/`.

The `userIgnoreFilters` field in `.obsidian/app.json` tells Obsidian which paths
to exclude from its vault index.

**Build the ignore list** from the following sources:
1. All entries in `{SOURCE_DIRS}` collected in Phase 2 — these are source directories
   that contain raw code, not wiki pages.
2. Standard non-wiki directories that should always be excluded:
   - `src`, `lib`, `assets`, `scripts`, `config`, `build`, `dist`, `out`,
     `target`, `node_modules`, `.git`, `.claude`, `.github`, `vendor`,
     `Pods`, `.gradle`, `coverage`, `tmp`, `temp`, `test`, `tests`,
     `spec`, `specs`, `__pycache__`, `.cache`
3. Any directory discovered in Phase 2 that was classified as source-like,
   asset-like, script-like, or config-like (all non-wiki directories).

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
array — adding any new entries from the list above without removing existing ones.
Write the merged file back.

Format each entry as a plain directory name without trailing slash (Obsidian convention).
For nested paths like `assets/shaders`, add both `assets/shaders` and `assets` as
separate entries so neither level appears in the graph.

---

### PHASE 7 — Final Report

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

- This skill never reads source file contents — only filenames, extensions, and
  build manifests (for language/build detection). Source content is left entirely
  to `/scan`.
- If a step fails (e.g., skill download fails due to no network), continue with
  the remaining steps and list failures in the final report. Never abort mid-run.
- The `{TODAY}` placeholder should be substituted with the actual current date in
  `YYYY-MM-DD` format wherever it appears in generated file content.
- The `{SOURCE_DIRS_FORMATTED}` placeholder produces a bulleted markdown list:
  ```
  - `src/` — source code (primary)
  - `lib/` — vendored libraries (index exports and public APIs only)
  - `assets/shaders/` — GLSL/HLSL shader files (index uniforms and stage)
  ```
- The `{SOURCE_DIRS_INLINE}` placeholder produces a comma-separated list for the log.
