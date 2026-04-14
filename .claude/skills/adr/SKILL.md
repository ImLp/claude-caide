---
name: adr
description: Interactively create a new Architectural Decision Record in wiki/adrs/. Guides you through context, decision, consequences, and files-to-modify. Cross-references wiki module pages automatically. Also supports updating existing ADR status. Examples: /adr Centralize CLI settings, /adr update ADR-001 status accepted
trigger: /adr
---

# /adr

## Purpose
Create a new Architectural Decision Record (ADR) in `wiki/adrs/`. ADRs document
significant architectural decisions, refactor plans, deprecations, and structural
changes — preserving the reasoning and consequences for future developers.

## Trigger
`/adr [title]`

Examples:
- `/adr Centralize CLI settings in config/cli.toml`
- `/adr Remove legacy EventBus and migrate to typed EventEmitter`
- `/adr Move water shaders to dedicated assets/shaders/water/ directory`
- `/adr` — no title, Claude will prompt for one

## Workflow

1. **Determine the ADR number.**
   Read `wiki/adrs/` to find the highest existing `adr_number`. Increment by 1.
   Format as zero-padded three digits: `001`, `002`, `023`, etc.

2. **Collect the title.**
   If a title was provided with the trigger, use it.
   If no title was provided, ask: "What is the architectural decision or change
   you want to document?"

3. **Gather the ADR content interactively.**
   Ask the user the following questions in sequence. Do not ask all at once —
   wait for each answer before asking the next:

   a. **Context:** "What is the current state of the codebase that makes this
      decision necessary? What problem are you solving?"

   b. **Decision:** "What exactly will you do? Describe the change specifically."

   c. **Consequences:** "What are the consequences? Which files or modules need
      to change? What new constraints does this create for future work?"

   d. **Files to modify:** "List the specific files that will need to change.
      (I will cross-reference these against the wiki.)"

   e. **Status:** "What is the current status of this decision?
      (proposed | accepted | superseded | deprecated)" — default: `proposed`.

4. **Cross-reference with the wiki.**
   For each file listed in "files to modify":
   - Check if a corresponding `wiki/modules/` page exists.
   - If yes, include the wiki link in `related_modules:`.
   - If no, note: "wiki page missing — run `/scan {path}` after implementing."

5. **Create the ADR file.**
   Write to `wiki/adrs/adr-{NNN}-{slugified-title}.md`.
   Use this frontmatter schema:
   ```yaml
   ---
   type: adr
   adr_number: "{NNN}"
   title: "{Title}"
   status: proposed | accepted | superseded | deprecated
   decision_date: {YYYY-MM-DD}
   context: |
     {Multi-line context from user input}
   decision: |
     {Multi-line decision from user input}
   consequences:
     - "{consequence 1}"
     - "{consequence 2}"
   files_to_modify:
     - {src/path/to/file.ext}
   related_modules:
     - [[module-link]]
   supersedes: null
   superseded_by: null
   ---
   ```
   Body: Expand on the decision with any additional context, diagrams in ASCII,
   or step-by-step implementation plan if the user provided one.

6. **Update `wiki/index.md`.**
   Add the new ADR under the `## Architectural Decision Records` section:
   ```
   - [[adr-{NNN}-{slug}]] — {STATUS} — {decision_date} — {title}
   ```

7. **Append to `wiki/log.md`.**
   ```
   ## [{YYYY-MM-DD}] adr | ADR-{NNN}: {title}
   Status: {proposed | accepted | ...}
   Files to modify: {N}
   Related modules: {comma-separated wiki links}
   ```

8. **Confirm to the user.**
   Report:
   - The ADR file path created.
   - Its number and status.
   - Any modules referenced that lack wiki pages (prompt to run `/scan`).
   - Suggest: "When this decision is implemented, run `/scan {path}` on the
     modified files to update the wiki, then mark this ADR as `accepted`."

## Updating an Existing ADR

To update an existing ADR's status:
```
/adr update ADR-001 status accepted
/adr update ADR-002 status superseded by ADR-005
```

When the user provides an update command:
1. Read the existing ADR file.
2. Update the `status:` field.
3. If superseded: set `superseded_by:` on the old ADR and `supersedes:` on the new one.
4. Update the entry in `wiki/index.md`.
5. Append to `wiki/log.md`.

## Notes
- ADRs are the only wiki pages that should be hand-edited by humans after creation.
  All other wiki pages are maintained by scans.
- Never delete an ADR. Supersede or deprecate it instead — the history of why
  decisions were made is as valuable as the decisions themselves.
- If the user says "we decided to..." or "we're going to refactor..." during any
  conversation, proactively suggest filing an ADR.
