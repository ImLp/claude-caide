---
name: trace
description: Reconstruct the full dependency chain for a module — upstream (what it depends on) and downstream (what depends on it) — up to 3 levels deep. Produces a structured map with impact ratings. Use before any architectural query or refactor. Examples: /trace EventBus, /trace src/core/EventBus.cpp
trigger: /trace
---

# /trace

## Purpose
Reconstruct the full dependency chain for a given module — both upstream (what it
depends on) and downstream (what depends on it). Produces a navigable map of how
the module is connected to the rest of the codebase.

## Trigger
`/trace [module]`

Examples:
- `/trace EventBus` — trace by component/module name
- `/trace src/core/EventBus.cpp` — trace by file path
- `/trace wiki/modules/event-bus.md` — trace by wiki page path

## Workflow

1. **Resolve the target.**
   Accept a module name, file path, or wiki page path as input.
   - If a name is given, search `wiki/index.md` for the matching module entry.
   - If no exact match, list candidates and ask the user to confirm.
   - Read the target `wiki/modules/{module}.md` page.

2. **Trace upstream dependencies (what this module depends on).**
   Read the `imports:` field of the target module page.
   For each import:
   - Read its `wiki/modules/` page.
   - Read the corresponding `wiki/dependencies/` page linking the two.
   - Recursively trace one level deeper (imports of imports).
   Stop at 3 levels deep or when reaching modules with no further imports.

3. **Trace downstream consumers (what depends on this module).**
   Search `wiki/dependencies/` for all pages where `from:` or `to:` references
   the target module.
   For each consumer found:
   - Read its `wiki/modules/` page.
   - Note the relationship type and impact rating.
   - Recursively trace one level deeper (consumers of consumers).
   Stop at 3 levels deep or when reaching modules with no further consumers.

4. **Produce the dependency map.**
   Output a structured report in this format:

   ```
   TRACE REPORT: {ModuleName}
   Source: {src_path}

   ── UPSTREAM (what {module} depends on) ──
   Level 1 (direct imports):
     → [[module-a]] (imports, HIGH impact)
     → [[module-b]] (imports, MEDIUM impact)
   Level 2 (transitive):
     → [[module-c]] via [[module-a]]
     → [[module-d]] via [[module-b]]
   Level 3:
     (none / or list)

   ── DOWNSTREAM (what depends on {module}) ──
   Level 1 (direct consumers):
     ← [[module-x]] (imports, HIGH impact)
     ← [[module-y]] (calls, MEDIUM impact)
   Level 2 (transitive):
     ← [[module-z]] via [[module-x]]
   Level 3:
     (none / or list)

   ── SUMMARY ──
   Total upstream dependencies: N (direct: N, transitive: N)
   Total downstream consumers:  N (direct: N, transitive: N)
   Architectural impact rating: LOW | MEDIUM | HIGH | CRITICAL
   ```

   Impact rating scale:
   - CRITICAL: 14+ total consumers
   - HIGH: 8–13 total consumers
   - MEDIUM: 3–7 total consumers
   - LOW: 0–2 total consumers

5. **Flag risks.**
   After the map, note any of the following if detected:
   - Circular dependencies (A → B → A).
   - Modules with both high upstream and high downstream counts (architectural bottlenecks).
   - Consumers with `confidence: low` (dependency relationship may be incomplete).

6. **Offer follow-up actions.**
   - "Run `/impact {module}` to get a risk-rated modification report."
   - "Run `/adr Refactor {module} dependencies` to file an architectural decision."
   - "File this trace as a wiki dependency page."

## Notes
- If `wiki/dependencies/` pages are sparse or missing, trace directly from `imports:`
  fields in module pages and note that dependency pages should be created via `/scan`.
- Do not read raw `src/` files during trace — work exclusively from wiki pages.
  If a wiki page is missing, note the gap and suggest running `/scan` on that path.
