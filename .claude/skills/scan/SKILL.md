---
name: scan
description: Parse a source path and generate or update CAIDE wiki pages for all modules, dependencies, and components found within it. Use after any source code change to keep wiki/ synchronized with src/. Examples: /scan src/, /scan src/rendering/, /scan src/core/EventBus.cpp
trigger: /scan
---

# /scan

## Purpose
Parse a source path and generate or update the CAIDE wiki pages for all modules,
dependencies, and components found within it. This is the primary ingestion operation
that keeps `wiki/` synchronized with `src/`.

## Trigger
`/scan [path]`

Examples:
- `/scan src/` — full codebase scan
- `/scan src/rendering/` — subsystem scan
- `/scan src/rendering/WaterRenderer.cpp` — single file scan
- `/scan assets/shaders/` — scan a source-like directory

## Workflow

1. **Parse the path.**
   Read all source files in the specified path recursively. For each file, identify:
   - The module's role and responsibility (infer from class names, function names, comments).
   - All exports: public functions, classes, structs, constants, types.
   - All imports: `#include`, `import`, `require`, `use`, or equivalent for the language.

2. **Identify what has changed.**
   Compare against existing `wiki/modules/` pages:
   - New files → create new module pages.
   - Changed files → update existing module pages; note what changed.
   - Deleted files → mark corresponding wiki pages as `deprecated` in frontmatter.

3. **Create or update `wiki/modules/` pages.**
   One page per source file. Use this frontmatter schema:
   ```yaml
   ---
   type: module
   title: "{ModuleName}"
   src_path: {relative/path/to/file.ext}
   language: {cpp|js|ts|py|rs|go|...}
   exports:
     - {ExportedSymbol::methodSignature()}
   imports:
     - [[wiki-link-to-imported-module]]
   related: []
   created: {YYYY-MM-DD}
   updated: {YYYY-MM-DD}
   confidence: high | medium | low
   ---
   ```
   Body: 2–4 sentences describing the module's purpose and architectural role.

4. **Create or update `wiki/dependencies/` pages.**
   For every import relationship discovered:
   - Check if a dependency page already exists between these two modules.
   - If not, create `wiki/dependencies/{from}-to-{to}.md`.
   - If yes, update it with any new relationship details.
   Use this frontmatter schema:
   ```yaml
   ---
   type: dependency
   title: "Dependency: {A} → {B}"
   from: [[module-a]]
   to: [[module-b]]
   relationship: imports | inherits | instantiates | calls | configures
   direction: unidirectional | bidirectional
   impact: high | medium | low
   notes: "{Why this dependency exists. What breaks if removed.}"
   created: {YYYY-MM-DD}
   updated: {YYYY-MM-DD}
   ---
   ```

5. **Create or update `wiki/components/` pages.**
   For every significant class, struct, interface, or function discovered:
   - "Significant" means: exported publicly, consumed by multiple modules, or
     architecturally load-bearing (e.g., base classes, singletons, event dispatchers).
   - Skip private helper functions and internal implementation details.
   Use this frontmatter schema:
   ```yaml
   ---
   type: component
   component_type: class | struct | function | interface | shader | config
   title: "{ComponentName}"
   defined_in: [[module-name]]
   signature: "{full type signature or function signature}"
   role: "{1–2 sentence description of architectural role}"
   dependencies:
     - [[dep-link]]
   consumers:
     - [[module-link]]
   created: {YYYY-MM-DD}
   updated: {YYYY-MM-DD}
   confidence: high | medium | low
   ---
   ```

6. **Update `wiki/index.md`.**
   For every new or updated page, add or revise its entry under the appropriate
   section (Modules, Dependencies, Components). Format:
   ```
   - [[module-name]] — `src/path/to/file.ext` — One-line summary (N consumers)
   ```

7. **Append to `wiki/log.md`.**
   Use this format:
   ```
   ## [{YYYY-MM-DD}] scan | {path}
   Files scanned: {N}
   Pages created: {comma-separated wiki page names}
   Pages updated: {comma-separated wiki page names}
   Pages deprecated: {comma-separated wiki page names, or "none"}
   Orphans detected: {list or "none"}
   ```

## Output to User
After completing the scan, report:
- How many files were scanned.
- How many wiki pages were created, updated, and deprecated.
- Any orphaned files detected (files in `src/` that now have no wiki entry).
- Any dependency relationships that could not be resolved (dangling imports).
- Suggest next steps (e.g., "Run `/health` to check for broken links").

## Notes
- Set `confidence: low` on any module whose role could not be clearly inferred.
- If a file is very large (>500 lines), create the module page with a summary and
  note that a deeper component-level scan is recommended.
- Do not scan `node_modules/`, `.git/`, or `.claude/` directories.
- Respect the `## Source-Like Directories` list in `CLAUDE.md` when determining
  what counts as a scannable path.
