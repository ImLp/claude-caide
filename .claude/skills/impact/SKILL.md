---
name: impact
description: Produce a risk-rated impact report for a module before modifying, renaming, or removing it. Lists all direct and transitive consumers, rates blast radius (LOW/MEDIUM/HIGH/CRITICAL), and determines whether an ADR is required. Read-only — makes no changes. Examples: /impact EventBus, /impact src/core/EventBus.cpp
trigger: /impact
---

# /impact

## Purpose
Produce a risk-rated impact report for a given module before it is modified, renamed,
or removed. Identifies every direct and transitive consumer, assesses blast radius,
and determines whether an Architectural Decision Record is required before proceeding.

## Trigger
`/impact [module]`

Examples:
- `/impact EventBus` — assess impact by module name
- `/impact src/core/EventBus.cpp` — assess by file path
- `/impact WaterRenderer::render` — assess impact of a specific exported symbol

## Workflow

1. **Resolve the target.**
   Accept a module name, file path, or exported symbol as input.
   - If a module name or path: read `wiki/modules/{module}.md`.
   - If a specific symbol: search `wiki/components/` for the symbol, then read its
     `defined_in` module page.
   - If no exact match found in wiki, report the gap and suggest running `/scan` first.

2. **Enumerate direct consumers.**
   Search `wiki/dependencies/` for all pages where `to:` references the target module.
   For each consumer:
   - Read the consumer's `wiki/modules/` page.
   - Note the `relationship` type (imports, inherits, instantiates, calls, configures).
   - Note the `impact` rating on the dependency page.
   - List which specific exports of the target module this consumer uses
     (cross-reference `wiki/components/` pages if available).

3. **Enumerate transitive consumers.**
   For each direct consumer found in step 2:
   - Find their consumers in `wiki/dependencies/`.
   - Go 2 levels deep maximum (direct → level 2 → level 3).
   - Flag if the transitive chain is very deep (>3 levels) — this is an architectural risk.

4. **Assess symbol-level breakage.**
   If the target module has a `wiki/components/` page:
   - List every exported symbol (functions, classes, constants).
   - For each symbol, identify which consumers depend on it specifically.
   - Distinguish: "removing the module" vs. "removing/renaming this specific export."

5. **Produce the impact report.**

   ```
   IMPACT REPORT: {ModuleName}
   Source: {src_path}
   Generated: {YYYY-MM-DD}

   ── DIRECT CONSUMERS ({N} modules) ──
   1. [[module-x]] — relationship: imports — impact: HIGH
      Uses: ModuleName::methodA(), ModuleName::methodB()
   2. [[module-y]] — relationship: inherits — impact: HIGH
      Uses: ModuleName (base class)
   3. [[module-z]] — relationship: calls — impact: MEDIUM
      Uses: ModuleName::helperFn()

   ── TRANSITIVE CONSUMERS ──
   Level 2 (consumers of direct consumers):
     [[module-w]] via [[module-x]]
     [[module-v]] via [[module-y]]
   Level 3:
     [[module-u]] via [[module-w]]

   ── BLAST RADIUS ──
   Direct consumers:     {N}
   Transitive consumers: {N}
   Total affected:       {N}
   Overall impact:       LOW | MEDIUM | HIGH | CRITICAL

   ── SPECIFIC BREAKAGE IF REMOVED ──
   - [[module-x]] will fail to compile (missing import).
   - [[module-y]] loses base class — must be refactored to inherit elsewhere.
   - [[module-z]] call to helperFn() becomes undefined.

   ── ADR REQUIRED? ──
   {YES | NO} — {reason}
   ```

   Impact rating scale:
   - CRITICAL (14+ total affected) → ADR mandatory, team review required
   - HIGH (8–13) → ADR strongly recommended
   - MEDIUM (3–7) → ADR recommended for removals, optional for modifications
   - LOW (0–2) → proceed with standard code review

6. **Recommend next steps.**
   Based on the impact rating:
   - CRITICAL / HIGH: "Run `/adr {title}` to document this change before proceeding."
   - MEDIUM: "Consider running `/adr` if this is a removal or rename."
   - LOW: "Proceed. Standard code review is sufficient."
   - Always: "Run `/scan {path}` after making changes to update the wiki."

## Notes
- Work exclusively from wiki pages — do not read raw `src/` files during impact analysis.
  If wiki coverage is incomplete, note which modules lack pages and suggest `/scan` first.
- If a consumer's `wiki/modules/` page has `confidence: low`, flag the impact estimate
  as potentially incomplete.
- This skill does not make changes. It produces a report only.
