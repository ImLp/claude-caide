---
name: recall
description: Pre-load the most relevant wiki context before answering an architectural question or starting work on an unfamiliar subsystem. Uses qmd semantic search if available, falls back to wiki/index.md. Run at session start or when switching focus areas. Examples: /recall water rendering, /recall EventBus consumers
trigger: /recall
---

# /recall

## Purpose
Pre-load the most relevant wiki context before responding to a new architectural
question or starting work on an unfamiliar part of the codebase. Reduces
hallucination and ensures answers are grounded in scanned wiki knowledge rather
than inferred from training data.

## Trigger
`/recall [topic]`

Examples:
- `/recall water rendering` — pre-load context on the water rendering subsystem
- `/recall EventBus consumers` — pre-load dependency context for EventBus
- `/recall CLI configuration` — pre-load context before working on config refactor
- `/recall` — no topic, Claude will ask what area to load context for

## Workflow

### With qmd installed (recommended for large wikis)

1. **Run semantic search.**
   Execute: `qmd query "{topic}" --json`
   Parse the JSON response. Extract the top 5 file paths by relevance score.

2. **Read the top results.**
   Read each of the returned wiki pages in full.

3. **Summarize loaded context.**
   Report to the user (briefly — 3–5 bullet points):
   - Which pages were loaded.
   - Key architectural facts surfaced (module roles, critical dependencies, ADR status).
   - Any gaps or low-confidence pages detected in the loaded set.

4. **Proceed.**
   Answer the user's question or begin the requested task with the loaded context
   already active. Do not re-read pages already loaded.

### Without qmd (fallback — any wiki size)

1. **Search `wiki/index.md`.**
   Read `wiki/index.md`. Identify all entries that are relevant to the topic
   (by module name, path fragment, or keyword match in the one-line summary).

2. **Read identified pages.**
   Read up to 7 of the most relevant pages:
   - Prioritize: `wiki/modules/` pages directly named in the topic.
   - Then: `wiki/dependencies/` pages linking those modules.
   - Then: `wiki/components/` pages for key symbols.
   - Then: any `wiki/adrs/` with status `proposed` or `accepted` related to the topic.

3. **Check `wiki/hot.md`.**
   Read `wiki/hot.md`. If the current topic overlaps with the `### Current Focus`
   or `### Active Pages` sections, note this to the user — prior session context
   is already partially established.

4. **Summarize loaded context.**
   Report to the user (briefly):
   - Which pages were loaded.
   - Key architectural facts (module roles, dependency impact ratings, ADR status).
   - Any modules mentioned in the topic that lack wiki pages
     (suggest `/scan {path}` to create them).

5. **Proceed.**
   Answer the user's question or begin the requested task with loaded context active.

## When to Use

Run `/recall` at the start of any of these situations:
- Beginning work on a new subsystem you have not touched this session.
- Starting a query whose answer depends on dependency relationships.
- Before running `/impact` or `/trace` on a complex module.
- When switching focus areas mid-session (e.g., from rendering to audio).
- At the start of any session where `wiki/hot.md` does not already cover the topic.

## Notes
- `/recall` is a read-only operation. It does not write to any wiki pages.
- If qmd is not installed, the fallback (index.md search) works well for wikis
  under ~100 modules. Install qmd when the wiki grows larger.
- Token cost: loading 5–7 wiki pages costs approximately 2,000–5,000 tokens
  depending on page depth. This is the correct tradeoff — grounded answers
  are worth the context spend.
- If no relevant pages are found for the topic, report this and suggest running
  `/scan` on the relevant source path first.
