# CAIDE — Codebase Automated Intelligence & Documentation Engine

**Claude-CAIDE** is a build-ready blueprint for transforming any source code repository into a structured, searchable, and interconnected knowledge base — compatible with Obsidian — that enables deep architectural querying, dependency tracing, impact analysis, and frictionless developer onboarding.

> **How to use this document:** Share this entire document with Claude Code and say: *"Set up CAIDE for this repository based on this blueprint."* Read the Quick Start to bootstrap in minutes. Read the full document to understand every architectural decision.

---

## Table of Contents

1. [The Core Problem](#1-the-core-problem)
2. [The Three-Layer Architecture](#2-the-three-layer-architecture)
3. [Full Directory Tree](#3-full-directory-tree)
4. [CLAUDE.md — The Master Schema](#4-claudemd--the-master-schema)
5. [Core Operations: Scan, Query, Health](#5-core-operations-scan-query-health)
6. [Deployment Modes: Global vs. Local](#6-deployment-modes-global-vs-local)
7. [Obsidian Setup and Plugins](#7-obsidian-setup-and-plugins)
8. [Claude Code Integration Strategies](#8-claude-code-integration-strategies)
9. [Indexing and Logging](#9-indexing-and-logging)
10. [The Hot Cache Pattern](#10-the-hot-cache-pattern)
11. [Frontmatter Schemas and Dataview](#11-frontmatter-schemas-and-dataview)
12. [Search at Scale: qmd](#12-search-at-scale-qmd)
13. [Custom Slash Commands](#13-custom-slash-commands)
14. [Advanced: Self-Evolving Loop](#14-advanced-self-evolving-loop)
15. [Use Cases and Configuration Variants](#15-use-cases-and-configuration-variants)
16. [Best Practices and Failure Modes](#16-best-practices-and-failure-modes)
17. [Quick Start: 15-Minute Build](#17-quick-start-15-minute-build)
18. [Known Repos and Starter Kits](#18-known-repos-and-starter-kits)

---

## 1. The Core Problem

### The Knowledge Gap in Every Codebase

Every non-trivial codebase has a knowledge gap. The code exists. The intent behind it — the architectural decisions, the dependency chains, the rationale for structure — exists only in the minds of engineers who may have already left the team.

Traditional approaches fail:

- **Grep/search:** Finds text, not meaning. Cannot answer "what depends on this shader?"
- **Inline comments:** Decay immediately. Never updated on refactor.
- **Wikis (human-maintained):** Abandoned within weeks. Maintenance burden exceeds perceived value.
- **RAG over source files:** Rediscovers relationships on every query. No accumulation. No synthesis.

### The CAIDE Solution

CAIDE instructs Claude Code to act as a **persistent, disciplined wiki maintainer** for your codebase. Instead of querying raw source files at query time, Claude incrementally builds and maintains a structured `wiki/` layer — an interconnected set of markdown files that maps every module, dependency, component, and architectural decision across your `src/`.

This enables queries that would otherwise require hours of manual archeology:

> "/query Where can I find the water rendering shaders and what is reliant on them?"

> "Suggest the best location to move all command-line settings to a single centralized file and task out each file I must modify to reference it."

> "What would break if I remove `EventBus.js`?"

> "Which modules have no test coverage and what do they export?"

### The Paradigm: Compiled Knowledge

| Dimension | Grep / Manual Search | RAG over Source | CAIDE Wiki |
| :--- | :--- | :--- | :--- |
| **When knowledge is processed** | Query time, every time | Query time, every time | Scan time — once, incrementally |
| **Dependency graph** | Not available | Discovered ad-hoc | Pre-built, navigable |
| **Architectural intent** | Not captured | May hallucinate | Documented in ADRs |
| **Onboarding a new engineer** | Hours of exploration | Unreliable synthesis | Read `wiki/index.md` |
| **Impact analysis** | Requires expert knowledge | Guesswork | Explicit dependency links |
| **Who maintains it** | Human (burns out) | No one — resets each query | Claude (never bored) |

### Division of Labour

| Human | Claude Code |
| :--- | :--- |
| Trigger scans on changed paths | Parse modules and extract exports |
| Ask architectural questions | Trace dependency chains |
| Approve `src/` modifications | Update wiki pages on every scan |
| Decide what to refactor | Identify orphaned files and dead code |
| Think about product direction | Flag wiki/src divergence automatically |

---

## 2. The Three-Layer Architecture

### Layer 1 — Source Code (`src/`)

**Owner: Human (write). LLM: read and modify with permission. Rule: Always sync `wiki/` after modification.**

Your project's source code. This is the ground truth for what the software actually does. CAIDE treats `src/` as the authoritative input to the intelligence layer — not as an immutable archive. Claude may modify files in `src/` when explicitly instructed and must proactively identify which `wiki/` pages require updates as a consequence.

**Additional source-like directories** can be flagged in `CLAUDE.md` for indexing:
- `lib/` — third-party or vendored libraries worth documenting
- `assets/` — shaders, configuration files, binary assets with semantic meaning
- `scripts/` — build tooling, automation scripts
- `infra/` — infrastructure-as-code

### Layer 2 — The Intelligence Layer (`wiki/`)

**Owner: LLM (write and maintain). Human: read and explore.**

A directory of LLM-generated markdown files that form a structured knowledge graph of the codebase. Module pages, dependency maps, component documentation, and architectural decision records. The LLM creates pages, updates them when source changes arrive, maintains cross-references, and keeps the entire layer consistent with `src/`.

**You read it. Claude writes and maintains it.**

### Layer 3 — The Schema (`CLAUDE.md`)

**Owner: Human + LLM (co-evolved). Read by: Claude at every session start.**

A configuration document that defines how the wiki is structured, what conventions to follow, and what workflows to execute for each operation. Without it, every session resets to zero. With it, Claude inherits the entire operational context of the codebase instantly — knowing the module taxonomy, the scan workflow, the dependency conventions, and the safety rules.

> Start with the template in Section 4. Evolve it as your understanding of the codebase deepens.

---

## 3. Full Directory Tree

### Local Mode (wiki lives inside the repo)

```
my-project/                         ← Repo root, Claude Code working dir
│
├── CLAUDE.md                       ← Master schema (Layer 3). Read first.
│
├── src/                            ← Layer 1. Source code. Mutable with permission.
│   ├── core/
│   ├── rendering/
│   │   ├── shaders/
│   │   └── pipeline/
│   ├── systems/
│   └── utils/
│
├── lib/                            ← Vendored/third-party (flagged for indexing)
├── assets/                         ← Shaders, configs, static files (flagged for indexing)
├── scripts/                        ← Build tooling
│
├── wiki/                           ← Layer 2. LLM-owned intelligence layer.
│   ├── index.md                    ← Master catalog of all wiki pages (updated on every scan)
│   ├── log.md                      ← Append-only chronological operation record
│   ├── hot.md                      ← Rolling session hot cache (~500 words, read silently at start)
│   ├── overview.md                 ← High-level codebase synthesis and architecture summary
│   │
│   ├── modules/                    ← One page per source file or logical module
│   │   └── {module-name}.md
│   │
│   ├── dependencies/               ← Pages mapping architectural relationships
│   │   └── {dep-slug}.md
│   │
│   ├── components/                 ← Pages for classes, structs, functions, subsystems
│   │   └── {component-name}.md
│   │
│   └── adrs/                       ← Architectural Decision Records
│       └── adr-{NNN}-{slug}.md
│
├── .claude/                        ← Claude Code config (not Obsidian vault content)
│   ├── settings.json               ← Agent settings, model prefs, hooks
│   └── skills/                     ← Custom slash commands (subdirectory + SKILL.md per command)
│       ├── scan/SKILL.md           → /scan [path]
│       ├── trace/SKILL.md          → /trace [module]
│       ├── impact/SKILL.md         → /impact [module]
│       ├── health/SKILL.md         → /health
│       ├── adr/SKILL.md            → /adr [title]
│       └── recall/SKILL.md         → /recall [topic]
│
└── .obsidian/                      ← Obsidian config (auto-generated, ignored by LLM)
    ├── app.json
    └── plugins/
```

### Key `.obsidian/app.json` settings

```json
{
  "userIgnoreFilters": [
    "node_modules/",
    ".git/",
    ".claude/"
  ],
  "newFileLocation": "folder",
  "newFileFolderPath": "wiki",
  "attachmentFolderPath": "wiki/assets"
}
```

> **Why ignore `.claude/`?** Skills and agent memory are LLM operational files — not codebase knowledge. Excluding them prevents graph and search pollution in Obsidian.

---

## 4. CLAUDE.md — The Master Schema

This is the most critical file in the project. It is Claude's persistent operating context — the difference between a disciplined, consistent codebase analyst and a generic chatbot that resets every session.

Paste this at the repo root as `CLAUDE.md`. Customize the `## Project` and `## Source-Like Directories` sections.

```markdown
# CAIDE — Master Schema

## Project
[REPLACE: Project name, language(s), build system, primary domain]
Example: "Aquatic rendering engine — C++17, CMake. Real-time water simulation for AAA games."

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
- `lib/` — vendored libraries (index exports and public APIs only)
- `assets/shaders/` — GLSL/HLSL shader files (index uniforms and stage)
- `scripts/` — build scripts (index entry points and arguments)
[ADD OR REMOVE entries to match your project structure]

## Page Conventions
Every wiki page MUST have YAML frontmatter. Use these schemas:

### Module/File Pages (wiki/modules/)
---
type: module
title: "Module Name"
src_path: src/rendering/WaterRenderer.cpp
language: cpp
exports:
  - WaterRenderer::render(scene, camera)
  - WaterRenderer::setTessellationLevel(int)
imports:
  - [[shader-water-surface]]
  - [[graphics-pipeline]]
  - [[scene-graph]]
related: [[water-simulation]], [[rendering-pipeline]]
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
title: "WaterRenderer"
defined_in: [[module-water-renderer]]
signature: "class WaterRenderer : public IRenderer"
role: "Orchestrates real-time water surface rendering via tessellation shaders"
dependencies:
  - [[graphics-pipeline]]
  - [[shader-water-surface]]
consumers:
  - [[module-scene-manager]]
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: high | medium | low
---

### Architectural Decision Records (wiki/adrs/)
---
type: adr
adr_number: "001"
title: "Centralize all CLI settings in config/cli.toml"
status: proposed | accepted | superseded | deprecated
decision_date: YYYY-MM-DD
context: "CLI flags are scattered across 12 files making global changes error-prone"
decision: "All CLI argument definitions moved to config/cli.toml; modules read
  from a single Config singleton"
consequences:
  - "12 files must be updated to remove direct argparse usage"
  - "New modules must register settings in config/cli.toml"
files_to_modify:
  - src/main.cpp
  - src/systems/InputSystem.cpp
  - src/rendering/RenderConfig.cpp
related_modules: [[module-main]], [[module-input-system]], [[module-render-config]]
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
When I say "/health" or "health":
1. Find orphaned files — files in src/ with no corresponding wiki/modules/ entry.
2. Find broken dependency links — [[wiki-links]] in dependency pages that
   reference non-existent modules.
3. Identify undocumented changes — files in src/ with a git mtime newer than
   their corresponding wiki/modules/ updated date.
4. Detect dead code candidates — modules with no inbound dependency links.
5. Detect stale documentation — wiki pages that reference exports no longer
   present in src/.
6. Append a health entry to wiki/log.md.

## Log Format
Each log entry MUST start with this prefix for parsability:
## [YYYY-MM-DD] {scan|query|health|adr} | {description}

Example:
## [2026-04-13] scan | src/rendering/shaders/
Files scanned: 8 GLSL shader files
Pages created: wiki/modules/shader-water-surface.md,
               wiki/modules/shader-caustics.md
Pages updated: wiki/dependencies/water-renderer-to-shaders.md,
               wiki/overview.md
Orphans detected: none

## Safety Rules
- NEVER delete wiki pages. Mark as deprecated in frontmatter instead.
- Always ensure wiki/ stays synchronized with src/.
- When modifying src/, proactively identify and suggest necessary updates
  to wiki/ documentation before proceeding.
- Always update wiki/index.md and wiki/log.md on every operation.
- When uncertain about a component's role or relationship, set confidence: low.
- Cross-reference all new module pages to at least 2 existing wiki pages.
```

---

## 5. Core Operations: Scan, Query, Health

These are the three primary operations. Each is a complete, repeatable workflow defined in `CLAUDE.md`.

### Operation 1: Scan

**Trigger:** A path is provided for analysis — either the full `src/` tree or a specific subdirectory after changes.

**Exact prompts:**
```
/scan src/
/scan src/rendering/shaders/
/scan lib/physics/
```

**What Claude does (as defined in `CLAUDE.md`):**
1. Parse all source files in the specified path.
2. Identify new, changed, or deleted modules.
3. Create or update `wiki/modules/` pages — file path, exports, imports, role.
4. Create or update `wiki/dependencies/` pages — relationship type, direction, impact.
5. Create or update `wiki/components/` pages for classes, functions, and structs.
6. Update `wiki/index.md`.
7. Append to `wiki/log.md`.

**Incremental vs. full scan:**
- **Full scan:** `/scan src/` — use at project bootstrap or after a major refactor.
- **Incremental scan:** `/scan src/rendering/` — use after targeted changes to a subsystem.
- **File-level scan:** `/scan src/rendering/WaterRenderer.cpp` — use after modifying a single file.

**Compounding value:** Each scan enriches the dependency graph. After the third scan of a subsystem, Claude has enough cross-reference context to answer impact analysis questions that would otherwise require expert knowledge.

---

### Operation 2: Query

**Trigger:** Any architectural question, impact analysis, or location query.

**Example prompts:**
```
/query Where can I find the water rendering shaders and what is reliant on them?
```
```
/query Suggest the best location to move all command-line settings to a single
centralized file and task out each file I must modify to reference it.
```
```
/query What would break if I delete EventBus.js?
```
```
/query Which modules in src/systems/ have no consumers?
```

**The compounding insight:** Good architectural analyses can be filed as ADRs. A refactor plan, an impact analysis, a structural recommendation — these are valuable and should not disappear into chat history.

**Follow-up to file an answer:**
```
File this as an ADR.
```

The wiki then accumulates both scanned knowledge (from `src/`) and reasoned knowledge (from queries). Both directions compound.

---

### Operation 3: Health

**Trigger:** Periodic codebase health check (recommended: after every significant merge or weekly).

**Exact prompt:**
```
/health
```

**What Claude produces:**

```
Codebase Health Report — 2026-04-13

ORPHANED FILES (2)
- src/utils/legacy_parser.cpp — no wiki/modules/ entry
  Recommendation: scan this file or verify it is intentionally excluded.

- src/systems/OldEventSystem.cpp — no wiki/modules/ entry
  Recommendation: likely dead code — confirm with git log, then deprecate.

BROKEN WIKI LINKS (1)
- wiki/dependencies/renderer-to-shadow-pipeline.md references
  [[module-shadow-v1]] which no longer exists.
  Recommendation: update link to [[module-shadow-renderer]] (renamed).

UNDOCUMENTED CHANGES (3)
- src/rendering/WaterRenderer.cpp modified 2026-04-11
  wiki/modules/water-renderer.md last updated 2026-03-28.
  Recommendation: re-scan this file.

DEAD CODE CANDIDATES (1)
- wiki/modules/debug-overlay.md has 0 inbound dependency links.
  Recommendation: confirm unused, then mark deprecated.

STALE DOCUMENTATION (1)
- wiki/components/water-renderer.md references
  WaterRenderer::setReflectionQuality() which no longer exists in src/.
  Recommendation: remove or update to setRenderQuality().
```

---

## 6. Deployment Modes: Global vs. Local

CAIDE supports two operational modes. Choose based on your team structure and whether wiki portability matters.

### Global Mode — Cross-Project Knowledge Aggregation

**Use case:** A developer or architect maintaining a wiki that spans multiple repositories.

**Structure:**
```
~/caide-vault/                      ← Central vault root, Claude Code working dir
│
├── CLAUDE.md                       ← Master schema covering all indexed projects
│
├── projects/                       ← One directory per repo (symlinked or cloned)
│   ├── my-engine/ → ~/dev/my-engine/src/
│   ├── my-tools/ → ~/dev/my-tools/src/
│   └── shared-lib/ → ~/dev/shared-lib/
│
└── wiki/
    ├── index.md                    ← Cross-project master catalog
    ├── log.md
    ├── hot.md
    ├── overview.md
    ├── modules/                    ← Modules from all indexed projects
    ├── dependencies/               ← Includes cross-project dependencies
    ├── components/
    └── adrs/
```

**Benefits:**
- Single query surface across all repositories.
- Cross-project dependency tracing (e.g., "which projects consume `shared-lib/EventBus`?").
- Architectural decisions visible across the entire engineering surface.

**Trade-offs:**
- Wiki is not versioned alongside the code it documents.
- Requires symlinks or git submodules to bridge repos.
- Less useful for onboarding a contributor to a single repo.

---

### Local Mode — Wiki Versioned Inside the Repository

**Use case:** A team that wants the wiki co-located with the source, versioned in the same commit history, and shareable with every contributor who clones the repo.

**Structure:**
```
my-project/                         ← Repo root
├── CLAUDE.md
├── src/
├── wiki/                           ← Committed to the repository
│   ├── index.md
│   ├── log.md
│   ├── hot.md
│   ├── overview.md
│   ├── modules/
│   ├── dependencies/
│   ├── components/
│   └── adrs/
└── .claude/
```

**Add to `.gitignore`:**
```
# CAIDE — never commit session state
wiki/hot.md
```

**Add to `.gitattributes`:**
```
wiki/** linguist-documentation=true
```

**Benefits:**
- Wiki evolves in lockstep with `src/` — every PR can include wiki updates.
- New contributors `git clone` and immediately have full architectural context.
- `git log wiki/` reveals the intellectual history of architectural decisions.
- ADRs in `wiki/adrs/` serve as the authoritative refactor changelog.

**Trade-offs:**
- Wiki noise in commit history if not managed with consistent commit conventions.
- PR review must include wiki update verification.

**Recommended CI check (Local Mode):**
```yaml
- name: Verify wiki sync
  run: |
    # Fail if any src/ file changed but its wiki/modules/ counterpart was not updated
    changed_src=$(git diff --name-only HEAD~1 | grep '^src/')
    for f in $changed_src; do
      module=$(basename $f | sed 's/\.[^.]*$//')
      if ! git diff --name-only HEAD~1 | grep -q "wiki/modules/$module"; then
        echo "WARNING: $f changed but wiki/modules/$module.md was not updated."
      fi
    done
```

---

## 7. Obsidian Setup and Plugins

Obsidian serves as the **visual IDE** for the wiki layer — graph view for dependency topology, Dataview for structured queries across frontmatter, and Templater for enforcing schema conventions.

### Vault Configuration Steps

1. Install Obsidian from [obsidian.md](https://obsidian.md).
2. Open the repository root as a new Obsidian vault.
3. Go to **Settings → Files and links**:
   - Set **Default location for new notes** → `wiki`
   - Set **Attachment folder path** → `wiki/assets`
4. Go to **Settings → Appearance** → enable **Readable line length** (OFF) — code snippets render better at full width.

### Essential Plugins

| Plugin | Type | Purpose |
| :--- | :--- | :--- |
| **Dataview** | Core | Query module frontmatter as a database; build dashboards |
| **Templater** | Core | Enforce consistent frontmatter schemas on new wiki pages |
| **Graph View** (built-in) | Core | Visualize dependency topology; identify hubs and orphans |
| **Kanban** | Optional | Track ADR status (proposed → accepted → superseded) |
| **File Explorer++** | Optional | Better folder management for large module trees |
| **Smart Connections** | Optional | Semantic similarity sidebar — find conceptually related modules |
| **Folder Note** | Optional | `index.md` per folder as a navigable root |

### Graph View Usage

After scans, use graph view to inspect dependency topology:
- **Hub nodes** (large, many connections) = architectural load-bearers — high-impact, high-risk to modify.
- **Orphan nodes** (isolated) = dead code candidates or documentation gaps.
- **Dense clusters** = tightly coupled subsystems — prime refactor candidates.
- **Bridge nodes** (few connections but between clusters) = critical integration points.

### Dataview Dashboards

**All modules by dependency count (architectural load):**
```dataview
TABLE length(imports) AS "Imports", length(file.outlinks) AS "Consumers", confidence, updated
FROM "wiki/modules"
SORT length(imports) DESC
```

**ADRs by status:**
```dataview
TABLE adr_number, status, decision_date
FROM "wiki/adrs"
SORT adr_number ASC
```

**Stale wiki pages (updated more than 30 days ago):**
```dataview
TABLE src_path, updated
FROM "wiki/modules"
WHERE updated < date(today) - dur(30 days)
SORT updated ASC
```

---

## 8. Claude Code Integration Strategies

| Strategy | Setup | Pros | Cons | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **1. Repo Root as Working Dir** | Open Claude Code at repo root | Simplest; `CLAUDE.md` auto-loaded; `src/` and `wiki/` in same context | Wiki and source in same space | Local Mode (recommended default) |
| **2. Symlinked Source** | `ln -s ~/dev/my-repo/src ./projects/my-repo` from central vault | Single query surface across projects | Symlink sync issues; recursive indexing risk | Global Mode with multiple repos |
| **3. MCP Bridge** | `obsidian-claude-code-mcp` plugin, WebSocket port 22360 | Clean separation; remote vault | More complex setup | Teams; vault on different machine |
| **4. One Vault Per Repo** | Separate Obsidian vault per project | Simple isolation | No cross-project dependency queries | Isolated project wikis |
| **5. qmd + Session Sync** | qmd MCP server + `/recall` skill | 60%+ token reduction on large wikis | Requires qmd install | Large codebases (200+ modules) |

> **Recommended for most users:** Strategy 1. Point Claude Code at the repository root. `CLAUDE.md` is auto-detected and loaded. Upgrade to Strategy 5 when `wiki/index.md` grows large enough to strain context.

---

## 9. Indexing and Logging

Two files are critical infrastructure for how Claude navigates the wiki. Both are defined in `CLAUDE.md` and updated on every operation.

### `wiki/index.md` — The Codebase Catalog

**Purpose:** Replace filesystem search with a human-readable, LLM-readable catalog of every indexed module, dependency, component, and architectural decision.

**Structure:**
```markdown
# CAIDE Index — [Project Name]

## Modules
- [[module-water-renderer]] — `src/rendering/WaterRenderer.cpp` — Tessellation-based water surface rendering (6 consumers)
- [[module-shader-water-surface]] — `assets/shaders/water_surface.glsl` — Vertex + fragment shader for water rendering
- [[module-event-bus]] — `src/core/EventBus.cpp` — Pub/sub event dispatcher; consumed by 14 modules

## Dependencies
- [[dep-water-renderer-to-shaders]] — WaterRenderer imports 3 shader modules (HIGH impact)
- [[dep-event-bus-fanout]] — EventBus consumed by 14 modules (CRITICAL — do not remove)

## Components
- [[comp-water-renderer]] — `class WaterRenderer : public IRenderer`
- [[comp-event-bus]] — `class EventBus<T>` — Generic typed event dispatcher

## Architectural Decision Records
- [[adr-001-centralize-cli-config]] — ACCEPTED — 2026-04-10 — Centralize CLI settings
- [[adr-002-remove-legacy-parser]] — PROPOSED — 2026-04-13 — Deprecate legacy_parser.cpp
```

**Rule:** Claude updates this file on EVERY scan. It reads it first on EVERY query to identify relevant pages.

### `wiki/log.md` — The Operation Timeline

**Purpose:** Chronological record of all scans, queries, health checks, and ADRs. Gives Claude context about recent changes when starting a new session.

```markdown
# Operation Log

## [2026-04-13] scan | src/rendering/shaders/
Files scanned: 8 GLSL shader files
Pages created: wiki/modules/shader-water-surface.md, wiki/modules/shader-caustics.md
Pages updated: wiki/dependencies/water-renderer-to-shaders.md
Orphans detected: none

## [2026-04-13] query | Impact analysis: remove EventBus.js
Pages read: modules/event-bus.md, 14 dependency pages
Output filed: wiki/adrs/adr-002-event-bus-impact.md

## [2026-04-10] health | Weekly health check
Orphans: 2 | Broken links: 1 | Undocumented changes: 3 | Dead code candidates: 1
```

**Tip:** The `## [YYYY-MM-DD]` prefix makes every entry greppable:
```bash
grep "^## \[" wiki/log.md | tail -10
```

---

## 10. The Hot Cache Pattern

`wiki/hot.md` solves the **context reset problem**: at the start of each new session, Claude has no memory of what architectural context was established previously.

### What It Is

`wiki/hot.md` is a ~500-word rolling session context file that Claude reads silently at every session start, before any other operation. It captures the current refactor focus, in-progress scans, open architectural questions, and the most recent operations — so Claude resumes with full context rather than starting cold.

### Add to Your CLAUDE.md

```markdown
## Hot Cache (`wiki/hot.md`)
Read `wiki/hot.md` silently at the start of EVERY session, before responding.
This file contains ~500 words of recent operational context. Do not summarize
it to the user — just use it to restore your working context.

After EVERY session (or when the user says /close), update wiki/hot.md:
- Keep total length under 500 words.
- Overwrite (do not append).
- Structure:

### Current Focus
[1–2 sentences: what architectural work is actively in progress]

### Open Questions
[Bullet list of unresolved architectural questions or planned scans]

### Recent Decisions
[Bullet list: key architectural conclusions from the last 1–2 sessions]

### Last Operations
[3–5 lines from wiki/log.md — most recent scan/query/health entries]

### Active Pages
[Wiki pages currently being developed or recently updated]
```

### Example `wiki/hot.md`

```markdown
---
type: hot-cache
updated: 2026-04-13
---

### Current Focus
Investigating dependency chain for water rendering subsystem. Evaluating
impact of centralizing CLI configuration as proposed in ADR-001.

### Open Questions
- Does WaterRenderer depend on GlobalConfig directly or via RenderConfig?
- Which modules in src/systems/ will require changes for ADR-001?
- Is legacy_parser.cpp referenced anywhere or truly dead?

### Recent Decisions
- ADR-001 (centralize CLI config) status changed to ACCEPTED — 2026-04-10
- wiki/modules/shader-caustics.md confidence set to low — incomplete exports
- EventBus identified as critical — 14 consumers, must not be removed without ADR

### Last Operations
[2026-04-13] scan | src/rendering/shaders/ — 8 files, 2 pages created
[2026-04-13] query | EventBus impact analysis → filed as ADR-002
[2026-04-10] health | 2 orphans, 1 broken link, 3 undocumented changes

### Active Pages
- wiki/modules/shader-water-surface.md (newly created, low confidence)
- wiki/adrs/adr-001-centralize-cli-config.md (accepted — tracking files to modify)
- wiki/adrs/adr-002-event-bus-impact.md (proposed — awaiting decision)
```

> **Token cost:** A 500-word hot cache costs ~650–700 tokens per session to read. This replaces 2,000–4,000 tokens of architectural recap. Net saving: ~3× per session.

> **Rule:** `wiki/hot.md` is the **only** wiki file Claude writes without explicit instruction. All other writes are triggered by scan, query, or health operations. Never commit `wiki/hot.md` — add it to `.gitignore`.

---

## 11. Frontmatter Schemas and Dataview

### Complete Frontmatter Schema Set

**Module/File Page (`wiki/modules/`)**
```yaml
---
type: module
title: "WaterRenderer"
src_path: src/rendering/WaterRenderer.cpp
language: cpp
exports:
  - "WaterRenderer::render(const Scene&, const Camera&)"
  - "WaterRenderer::setTessellationLevel(int level)"
  - "WaterRenderer::setReflectionQuality(ReflectionQuality)"
imports:
  - [[shader-water-surface]]
  - [[graphics-pipeline]]
  - [[global-config]]
related:
  - [[comp-water-renderer]]
  - [[dep-water-renderer-to-shaders]]
created: 2026-04-10
updated: 2026-04-13
confidence: high
---
```

**Dependency/Link Page (`wiki/dependencies/`)**
```yaml
---
type: dependency
title: "WaterRenderer → Shader Suite"
from: [[module-water-renderer]]
to:
  - [[module-shader-water-surface]]
  - [[module-shader-caustics]]
  - [[module-shader-foam]]
relationship: imports
direction: unidirectional
impact: high
notes: "WaterRenderer owns these shaders exclusively. Removal of any shader
  crashes the renderer at startup. Moving shaders requires updating
  WaterRenderer::loadShaders() and the CMakeLists.txt asset list."
created: 2026-04-10
updated: 2026-04-13
---
```

**Component/Class Page (`wiki/components/`)**
```yaml
---
type: component
component_type: class
title: "WaterRenderer"
defined_in: [[module-water-renderer]]
signature: "class WaterRenderer : public IRenderer"
role: "Orchestrates real-time water surface rendering. Manages shader loading,
  tessellation configuration, and per-frame draw call submission."
dependencies:
  - [[graphics-pipeline]]
  - [[shader-water-surface]]
  - [[global-config]]
consumers:
  - [[module-scene-manager]]
  - [[module-render-orchestrator]]
created: 2026-04-10
updated: 2026-04-13
confidence: high
---
```

**Architectural Decision Record (`wiki/adrs/`)**
```yaml
---
type: adr
adr_number: "001"
title: "Centralize CLI settings in config/cli.toml"
status: accepted
decision_date: 2026-04-10
context: "CLI argument definitions are scattered across 12 source files.
  Adding a new global flag requires modifying multiple files with no
  single authoritative location. This has caused multiple merge conflicts."
decision: "All CLI argument definitions will be moved to config/cli.toml.
  A Config singleton (src/core/Config.cpp) will parse this file at startup.
  All modules will read CLI values from Config:: rather than from argparse directly."
consequences:
  - "12 files must be updated to remove direct argparse usage"
  - "New modules must register settings in config/cli.toml only"
  - "Config singleton becomes a high-impact dependency — needs documentation"
files_to_modify:
  - src/main.cpp
  - src/systems/InputSystem.cpp
  - src/rendering/RenderConfig.cpp
  - src/core/Config.cpp (new file)
  - config/cli.toml (new file)
related_modules:
  - [[module-main]]
  - [[module-input-system]]
  - [[module-render-config]]
supersedes: null
superseded_by: null
---
```

### Dataview Queries

**High-impact modules (many consumers):**
```dataview
TABLE src_path, length(file.inlinks) AS "Consumers", confidence, updated
FROM "wiki/modules"
SORT length(file.inlinks) DESC
LIMIT 20
```

**All ADRs by status:**
```dataview
TABLE adr_number, status, decision_date
FROM "wiki/adrs"
WHERE type = "adr"
SORT adr_number ASC
```

**Low-confidence pages needing review:**
```dataview
TABLE title, src_path, updated
FROM "wiki"
WHERE confidence = "low"
SORT updated ASC
```

**Dead code candidates (no inbound links):**
```dataview
TABLE src_path, updated
FROM "wiki/modules"
WHERE length(file.inlinks) = 0
SORT updated ASC
```

**Recently modified modules (last 7 days):**
```dataview
LIST
FROM "wiki/modules"
WHERE updated >= date(today) - dur(7 days)
SORT updated DESC
```

---

## 12. Search at Scale: qmd

At small scale (~50 modules), `wiki/index.md` is sufficient. As the wiki grows past 200 modules, the index itself may strain the context window. This is when `qmd` is added.

### What qmd Is

`qmd` is a local, on-device search engine for markdown files built by Tobi Lütke (CEO of Shopify). It combines:

- **BM25 full-text search** — keyword precision for exact module names and export signatures.
- **Vector semantic search** — finds architecturally related pages even without keyword match.
- **LLM re-ranking** — highest quality; scores results by architectural relevance.

All models run locally via `node-llama-cpp` with GGUF models. No data leaves your machine.

### Install and Configure

```bash
npm install -g @tobilu/qmd

# Add the wiki as a named collection
qmd collection add ./wiki --name my-project

# Rebuild the index after scans
qmd index rebuild my-project
```

### Usage Patterns

```bash
# Keyword search for a specific module
qmd search "water renderer"

# Semantic search for architectural concepts
qmd vsearch "modules that handle real-time graphics pipeline state"

# Dependency tracing query (best quality)
qmd query "what depends on the EventBus and what would break if removed"

# JSON output for piping to Claude Code
qmd query "CLI configuration dependencies" --json

# Run as MCP server
qmd mcp
```

### Integration with Claude Code

The `/recall` skill is defined in `.claude/skills/recall/SKILL.md` (included with CAIDE).
It handles both the qmd path (when installed) and a fallback path using `wiki/index.md`
for wikis that do not have qmd configured.

> **Token savings:** Using qmd to pre-select relevant pages before loading reduces token usage by 60%+ compared to naively reading all modules or relying on large context windows.

---

## 13. Custom Slash Commands

All seven skills ship as ready-to-use files in `.claude/skills/`. Copy the entire
`.claude/skills/` directory into your target repository to install all commands.

```
.claude/skills/
├── caide-init/
│   └── SKILL.md    → /caide-init
├── scan/
│   └── SKILL.md    → /scan [path]
├── trace/
│   └── SKILL.md    → /trace [module]
├── impact/
│   └── SKILL.md    → /impact [module]
├── health/
│   └── SKILL.md    → /health
├── adr/
│   └── SKILL.md    → /adr [title]
└── recall/
    └── SKILL.md    → /recall [topic]
```

| Command | Trigger | Skill File | Workflow Summary |
| :--- | :--- | :--- | :--- |
| `/caide-init` | Project bootstrap | `caide-init/SKILL.md` | Interactive wizard: generate CLAUDE.md, scaffold wiki/, install skills, configure .obsidian. |
| `/scan [path]` | Changed source | `scan/SKILL.md` | Parse path, create/update modules/deps/components pages, update index, log. |
| `/trace [module]` | Dependency query | `trace/SKILL.md` | Reconstruct full upstream + downstream dependency chain with impact rating. |
| `/impact [module]` | Pre-refactor | `impact/SKILL.md` | Enumerate all consumers, assess blast radius, determine if ADR is required. |
| `/health` | Weekly maintenance | `health/SKILL.md` | Find orphans, broken links, undocumented changes, dead code, stale docs. |
| `/adr [title]` | Architectural decision | `adr/SKILL.md` | Interactive ADR creation with cross-referencing against wiki module pages. |
| `/recall [topic]` | Session prep | `recall/SKILL.md` | Pre-load top relevant wiki pages; grounded context before answering. |

### Command Reference

**`/caide-init`** — One-time project bootstrap wizard. Interactively collects project
identity (or auto-discovers languages, build system, and domain), discovers source-like
directories (respecting `.gitignore` / `.claudeignore`), installs any missing CAIDE
skills, generates `CLAUDE.md`, scaffolds `wiki/`, and writes `.obsidian/app.json` with
`userIgnoreFilters` set so only wiki pages appear in Obsidian's connections graph.

**`/scan [path]`** — The primary ingestion command. Run after any source change.
Parses source files, creates module/dependency/component wiki pages, updates
`wiki/index.md`, and appends to `wiki/log.md`.

**`/trace [module]`** — Reconstructs the full dependency graph around a module:
upstream (what it depends on) and downstream (what depends on it), up to 3 levels
deep. Outputs a structured map with impact ratings. Use before any architectural query.

**`/impact [module]`** — Pre-refactor safety check. Lists every direct and transitive
consumer of a module, rates the overall blast radius (LOW/MEDIUM/HIGH/CRITICAL), and
recommends whether an ADR is required before proceeding. Read-only — makes no changes.

**`/health`** — Full wiki health diagnostic. Detects: orphaned source files,
broken `[[wiki-links]]`, source files modified after their wiki page was last updated,
dead code candidates (zero inbound links), stale exported symbol documentation, and
low-confidence pages. Appends a summary to `wiki/log.md`.

**`/adr [title]`** — Interactive ADR creation. Guides you through context, decision,
consequences, and files-to-modify. Cross-references against existing wiki module pages.
Files the completed ADR in `wiki/adrs/adr-{NNN}-{slug}.md` and updates `wiki/index.md`.
Also supports `/adr update ADR-001 status accepted` to update existing ADR status.

**`/recall [topic]`** — Session context pre-loader. Uses qmd semantic search (if
installed) or `wiki/index.md` fallback to identify and read the most relevant wiki pages
before answering a question or starting work on an unfamiliar area. Always run this
when switching focus areas or starting a new session on a complex subsystem.

---

## 14. Advanced: Self-Evolving Loop

The CAIDE compounding loop has three entry points. Once all three operate consistently, the wiki improves automatically as the codebase evolves.

```
Source Code Changes         Architectural Queries        Periodic Maintenance
        │                          │                             │
        ▼                          ▼                             ▼
[/scan src/changed/]       [Questions / Queries]           [/health]
        │                          │                             │
        ▼                          ▼                             │
[SCAN operation]         [QUERY operation]                       │
        │                          │                             │
        └──────────┬───────────────┘                             │
                   ▼                                             │
           [wiki/ pages updated]                                 │
           - modules/ created/updated                            │
           - dependencies/ mapped                                │
           - components/ documented                              │
           - adrs/ filed from queries                            │
                   │                                             │
                   ▼                                             │
          [wiki/index.md updated]                                │
          [wiki/log.md appended]                                 │
                   │                                             │
                   ▼                                             │
     [Next query uses richer wiki] ──────────────────────────→  ▲
```

### Self-Evolving Review Prompt

Run this monthly for deep wiki maintenance:

```
Review the entire wiki for:
1. Modules in src/ with no wiki/modules/ entry — create stubs.
2. Wiki pages referencing exports that no longer exist — mark stale.
3. Dependency pages with broken links — update or deprecate.
4. Components mentioned across 5+ module pages but lacking their own
   wiki/components/ entry — create them.
5. Identify the 3 most architecturally risky areas of the codebase
   (high consumer count, low confidence, or recent undocumented changes).
6. Suggest 3–5 refactor opportunities worth raising as ADRs.
File a summary as wiki/adrs/adr-{NNN}-monthly-review-{date}.md.
```

### Claude Code Hooks (Automation)

Add to `.claude/settings.json` to trigger wiki updates after source file saves:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Source modified — run: scan {file} to update wiki'"
          }
        ]
      }
    ]
  }
}
```

> **Warning:** Fully automated scan-on-save bypasses the human review step. Use it for notification only (as above). Trigger actual scans explicitly to maintain wiki quality.

---

## 15. Use Cases and Configuration Variants

### A. Game Engine / AAA Game Project

**Config additions:**
- Flag `assets/shaders/` as a source-like directory for indexing.
- Add `shader_stage: vertex | fragment | compute | geometry` frontmatter to module pages for shader files.
- Add `wiki/subsystems/` for major engine subsystems (rendering, physics, audio, input).
- Lint check: "Are all shader uniforms documented in their module page?"

**High-value queries:**
- "What shaders are loaded by the water rendering subsystem?"
- "Which systems write to the global transform buffer?"

---

### B. Large Backend Service / Microservices

**Config additions:**
- Add `endpoint` frontmatter to module pages for files that expose HTTP routes.
- Add `wiki/contracts/` for API contract pages (request/response schemas).
- ADR conventions for breaking API changes.
- Lint check: "Are all public endpoints documented with their consumers?"

**High-value queries:**
- "Which services consume the UserService auth endpoint?"
- "What would break if we change the PaymentEvent schema?"

---

### C. Legacy Codebase Archeology

**Config additions:**
- Add `legacy: true` frontmatter flag to module pages for files identified as legacy.
- Add `wiki/migration/` for migration plan pages.
- Aggressive use of dead code detection in health workflow.

**High-value queries:**
- "Which modules have no consumers and were last modified before 2022?"
- "What is the minimum set of files I must touch to replace OldAuthSystem?"

---

### D. Open Source Library

**Config additions:**
- Add `public_api: true` frontmatter flag to module pages for exported public symbols.
- Add `wiki/changelog/` for version-tagged change summaries.
- Add `semver_impact: major | minor | patch` to ADRs.
- `wiki/` committed and published alongside the code (Local Mode).

**High-value queries:**
- "Which public API symbols have changed since v2.0.0?"
- "What is the dependency graph for the public streaming interface?"

---

### E. Team Onboarding

**Config additions:**
- Add `wiki/onboarding/` with guided reading paths through the wiki.
- Add `complexity: low | medium | high` frontmatter to module pages.
- Add `entry_points: true` flag to module pages for the primary files a new developer should read first.

**High-value queries:**
- "Give me a reading order for the rendering subsystem, starting with the simplest modules."
- "What are the 10 most important modules a new backend engineer should understand first?"

---

## 16. Best Practices and Failure Modes

### The Golden Rules

1. **`src/` is the authority. `wiki/` is the map.**
   If they disagree, `src/` wins. Never hand-edit wiki pages to paper over source inconsistencies — fix the source, then re-scan.

2. **Always keep `wiki/` synchronized with `src/`.**
   When modifying `src/`, proactively identify which wiki pages are affected and update them. An unsynchronized wiki is worse than no wiki — it misleads.

3. **One scan per logical change, not one scan per file.**
   Scanning a subsystem after completing a feature produces richer cross-references than scanning individual files piecemeal.

4. **Schema is persistent memory. Evolve it.**
   Start with the template in Section 4. After every 30–50 scans, review `CLAUDE.md`: add new page types you discovered you needed, refine workflows, document exceptions.

5. **Use modular `CLAUDE.md`, not a monolith.**
   As the schema grows, split it: `CLAUDE-scan.md`, `CLAUDE-query.md`, `CLAUDE-schema.md`. Reference them from the root. This keeps each section focused and easier to evolve.

6. **Commit `wiki/` alongside `src/` changes (Local Mode).**
   ```bash
   git add src/ wiki/ && git commit -m "feat: add tessellation control to WaterRenderer

   wiki: updated modules/water-renderer.md, components/water-renderer.md,
         dependencies/water-renderer-to-shaders.md"
   ```

7. **ADRs are first-class citizens.**
   Every significant architectural decision — refactors, deprecations, new patterns — should be an ADR. They are the institutional memory that survives team turnover.

### Common Failure Modes

| Failure Mode | Symptom | Fix |
| :--- | :--- | :--- |
| **No schema file** | Claude reinvents conventions every session | Write and maintain `CLAUDE.md` |
| **Monolithic CLAUDE.md** | Schema grows to 5,000+ tokens; Claude ignores parts | Split into modular sub-files |
| **Infrequent health checks** | Wiki drifts from `src/`; orphans and stale docs accumulate | Run health weekly or post-merge |
| **Skipping index updates** | Index drifts from reality; query quality degrades | Enforce index update in CLAUDE.md scan workflow |
| **Committing `wiki/hot.md`** | Session state noise in git history | Add `wiki/hot.md` to `.gitignore` |
| **No git history** | Bad scan corrupts pages; no rollback | `git init` at repo root on day one |
| **Oversized `wiki/index.md`** | Index hits context limit on large codebases | Add qmd; switch to semantic pre-loading |
| **Hand-editing wiki pages** | Wiki diverges from scanned reality | Use scans to update — edit only ADRs manually |
| **No ADR discipline** | Architectural decisions lost to chat history | File every significant decision as an ADR |

### When NOT to Use CAIDE

- You need real-time, sub-second dependency search across millions of files → use a dedicated language server (LSP) or static analysis tool.
- The team has no git discipline → file chaos defeats the compounding effect.
- The codebase changes faster than scans can keep up → consider triggering scans from CI rather than manually.
- You need a zero-maintenance system → CAIDE requires periodic health checks and scan triggers. Claude handles wiki maintenance, but you must trigger the operations.

---

## 17. Quick Start: 15-Minute Build

### Step 1 — Provision the Wiki Directory

```bash
# From your existing repository root
mkdir -p wiki/{modules,dependencies,components,adrs}
mkdir -p .claude/skills

touch wiki/index.md
touch wiki/log.md
touch wiki/overview.md
```

### Step 2 — Configure Obsidian

```bash
mkdir -p .obsidian
cat > .obsidian/app.json << 'EOF'
{
  "userIgnoreFilters": ["node_modules/", ".git/", ".claude/"],
  "newFileLocation": "folder",
  "newFileFolderPath": "wiki",
  "attachmentFolderPath": "wiki/assets"
}
EOF
```

Open the repository root in Obsidian as a new vault. Install the **Dataview** and **Templater** plugins.

### Step 3 — Create CLAUDE.md

Paste the full `CLAUDE.md` schema from Section 4 into the repository root as `CLAUDE.md`.

Customize:
- `## Project` — project name, language(s), build system, domain.
- `## Source-Like Directories` — any directories beyond `src/` to index.

### Step 4 — Update .gitignore

```bash
echo "wiki/hot.md" >> .gitignore
```

### Step 5 — Initialize Claude Code

```bash
# Ensure Claude Code is pointing at the repo root
claude init
```

### Step 6 — Bootstrap Prompt

Open Claude Code in the repository root and run:

```
Here is my CLAUDE.md schema. Please:
1. Read it fully.
2. Confirm you understand the directory structure and all three operations.
3. Initialize wiki/index.md with the correct header and empty category sections.
4. Initialize wiki/log.md with the log format header.
5. Initialize wiki/hot.md with the correct structure (Current Focus, Open Questions,
   Recent Decisions, Last Operations, Active Pages). Mark it as freshly initialized.
6. Write a brief wiki/overview.md describing the project domain and that the wiki
   is newly initialized.
7. Tell me which path in src/ to scan first.
```

### Step 7 — First Scan

```
/scan src/
```

Or, for large codebases, start with the core:
```
/scan src/core/
```

### Step 8 — Verify and Commit

Open Obsidian. Check:
- `wiki/modules/` has new module pages.
- `wiki/index.md` has been populated.
- `wiki/log.md` has a scan entry.
- Graph view shows the modules with dependency connections.

```bash
git add wiki/ CLAUDE.md && git commit -m "chore: initialize CAIDE wiki

Add wiki/ intelligence layer with index, log, and module pages
generated by initial scan of src/core/."
```

### Step 9 — First Architectural Query

```
Where do all the configuration values for the rendering pipeline come from?
Cite wiki pages. If this reveals a refactor opportunity, file an ADR.
```

The wiki is now live and compounding.

---

## 18. Known Repos and Starter Kits

| Resource | Type | Notes |
| :--- | :--- | :--- |
| `ScrapingArt/Karpathy-LLM-Wiki-Stack` | GitHub repo | Original LLM Wiki blueprint this project was forked from |
| `SamurAIGPT/llm-wiki-agent` | GitHub repo | Full agent implementation (adapt for CAIDE page types) |
| `huytieu/COG-second-brain` | GitHub repo | Self-evolving template with hooks |
| `ksanderer/claude-vault` | GitHub repo | Git-based cloud sync variant |
| Karpathy's original LLM Wiki gist | Gist | Foundational pattern: `gist.github.com/karpathy/442a6bf555914893e9891c11519de94f` |
| qmd by Tobi Lütke | npm package | `@tobilu/qmd` — local semantic search for markdown |
| Kepano's obsidian-skills | GitHub repo | Agent skill set for correct Obsidian syntax |

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                        DEVELOPER                            │
│  Triggers: scan / query / health / adr                      │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    CLAUDE CODE + CLAUDE.md                  │
│  Reads schema → executes workflow → updates wiki            │
└──────────┬────────────────────────────────────┬─────────────┘
           │                                    │
           ▼                                    ▼
┌──────────────────────┐            ┌───────────────────────────┐
│       src/           │            │         wiki/             │
│  (Layer 1 — Source)  │◄──scan─────│  (Layer 2 — Intelligence) │
│                      │            │                           │
│  WaterRenderer.cpp   │            │  modules/                 │
│  EventBus.cpp        │────────────►  dependencies/            │
│  shaders/*.glsl      │            │  components/              │
│  Config.cpp          │            │  adrs/                    │
│                      │            │  index.md                 │
│  [mutable]           │            │  log.md                   │
└──────────────────────┘            │  hot.md                   │
                                    │  overview.md              │
                                    │                           │
                                    │  [LLM-owned]              │
                                    └───────────────────────────┘
                                               │
                                               ▼
                                    ┌───────────────────────────┐
                                    │        OBSIDIAN           │
                                    │  Graph view, Dataview,    │
                                    │  ADR kanban, search       │
                                    └───────────────────────────┘
```
