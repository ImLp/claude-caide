---
name: health
description: Run a full CAIDE wiki health check. Detects orphaned source files, broken wiki links, undocumented source changes, dead code candidates, and stale documentation. Run after significant merges or weekly. Appends a summary to wiki/log.md.
trigger: /health
---

# /health

## Purpose
Run a full codebase health check on the CAIDE wiki. Detects divergence between
`src/` and `wiki/`, identifies dead code candidates, finds broken links, and
surfaces stale documentation. Produces an actionable health report.

## Trigger
`/health`

Run this: after every significant merge, weekly during active development,
or any time you suspect the wiki has drifted from the source.

## Workflow

1. **Find orphaned source files.**
   Enumerate all source files in `src/` (and any source-like directories listed
   in `CLAUDE.md`). For each file, check whether a corresponding `wiki/modules/`
   page exists (match by `src_path` frontmatter field).
   - Files with no wiki page = **orphans**.
   - Report each orphan with its path and last-modified date.

2. **Find broken wiki dependency links.**
   Read all pages in `wiki/dependencies/`. For every `from:` and `to:` field
   that contains a `[[wiki-link]]`, verify the referenced page exists in `wiki/`.
   - Links pointing to non-existent pages = **broken links**.
   - Report each broken link with the dependency page it was found in.

3. **Find undocumented source changes.**
   For every `wiki/modules/` page, compare its `updated:` frontmatter date against
   the last-modified date of its `src_path` file on disk.
   - Files where `src/` mtime is newer than `wiki/` `updated:` date = **undocumented changes**.
   - Report each case with both dates and suggest running `/scan {path}`.

4. **Find dead code candidates.**
   Read all `wiki/modules/` pages. For each module, count how many other wiki pages
   link to it (inbound links from `wiki/dependencies/` `from:` fields, or `imports:`
   fields in other module pages).
   - Modules with 0 inbound links = **dead code candidates**.
   - Report each candidate with its `src_path` and `updated:` date.
   - Note: inbound links = 0 in the wiki does not guarantee dead code — the wiki may
     be incomplete. Flag for human verification.

5. **Find stale documentation.**
   For every `wiki/components/` and `wiki/modules/` page, check whether the exports
   listed in `exports:` frontmatter still exist in the actual `src/` file.
   - Exported symbols in the wiki that are absent from `src/` = **stale documentation**.
   - Report each stale entry with the wiki page and missing symbol.

6. **Find low-confidence pages.**
   List all wiki pages where `confidence: low` is set. These are pages where Claude
   was uncertain during scanning and may require human review or a re-scan.

7. **Produce the health report.**

   ```
   CODEBASE HEALTH REPORT — {YYYY-MM-DD}
   ════════════════════════════════════════

   ORPHANED SOURCE FILES ({N})
   Files in src/ with no wiki/modules/ entry:
   - src/utils/legacy_parser.cpp (last modified: {date})
     → Suggest: /scan src/utils/legacy_parser.cpp OR confirm intentional exclusion.
   [... or "None detected."]

   BROKEN WIKI LINKS ({N})
   [[wiki-links]] that reference non-existent pages:
   - wiki/dependencies/renderer-to-shadow-v1.md → references [[module-shadow-v1]] (does not exist)
     → Suggest: update link to [[module-shadow-renderer]] or delete page if obsolete.
   [... or "None detected."]

   UNDOCUMENTED SOURCE CHANGES ({N})
   src/ files modified more recently than their wiki/modules/ page:
   - src/rendering/WaterRenderer.cpp (src modified: {date}, wiki updated: {date})
     → Suggest: /scan src/rendering/WaterRenderer.cpp
   [... or "None detected."]

   DEAD CODE CANDIDATES ({N})
   Modules with no inbound wiki dependency links:
   - wiki/modules/debug-overlay.md → src/debug/DebugOverlay.cpp (updated: {date})
     → Suggest: verify with git log, confirm unused, then deprecate or remove.
   [... or "None detected."]

   STALE DOCUMENTATION ({N})
   Wiki pages referencing exports no longer present in src/:
   - wiki/components/water-renderer.md references WaterRenderer::setReflectionQuality()
     which is absent from src/rendering/WaterRenderer.cpp.
     → Suggest: update export to setRenderQuality() or remove if deleted.
   [... or "None detected."]

   LOW-CONFIDENCE PAGES ({N})
   Pages requiring human review or re-scan:
   - wiki/modules/shader-caustics.md (confidence: low — incomplete exports detected)
   [... or "None detected."]

   ── SUMMARY ──
   Orphans:                {N}
   Broken links:           {N}
   Undocumented changes:   {N}
   Dead code candidates:   {N}
   Stale documentation:    {N}
   Low-confidence pages:   {N}
   Overall wiki health:    CLEAN | MINOR DRIFT | MODERATE DRIFT | SIGNIFICANT DRIFT
   ```

   Health rating:
   - CLEAN: all zeroes.
   - MINOR DRIFT: 1–3 issues total, no broken links.
   - MODERATE DRIFT: 4–10 issues, or any broken links.
   - SIGNIFICANT DRIFT: 10+ issues, or multiple stale exports, or many orphans.

8. **Append to `wiki/log.md`.**
   ```
   ## [{YYYY-MM-DD}] health | Weekly health check
   Orphans: {N} | Broken links: {N} | Undocumented changes: {N}
   Dead code candidates: {N} | Stale docs: {N} | Low confidence: {N}
   Overall: {CLEAN | MINOR DRIFT | MODERATE DRIFT | SIGNIFICANT DRIFT}
   ```

9. **Recommend next steps.**
   Based on the report, suggest:
   - Which paths to re-scan (`/scan`).
   - Which broken links to fix manually.
   - Which dead code candidates to verify with `git log`.
   - Whether a monthly deep review (`/health` + manual wiki review) is due.

## Notes
- This skill reads `wiki/` and `src/` but does NOT write any wiki pages.
  It is a read-only diagnostic. All fixes are triggered separately.
- If the wiki is newly initialized and most checks return "None detected," that is
  expected — run `/scan src/` first to populate the wiki before running `/health`.
