# Context-Management Workflow (ACTIVE_CONTEXT.md + session skills)

## Context

Sessions on this project grow long and hit `/compact`, and unrelated tasks force `/clear`. Each fresh start currently re-derives state by re-reading the repo, which burns context and reintroduces bloat. The goal is a lightweight, repeatable rhythm that carries just-enough state across the `/clear` and `/compact` boundaries:

- **CLAUDE.md** = permanent operating rules (already exists).
- **docs/ACTIVE_CONTEXT.md** = short session memory (NEW): current state, last task, known issues, next step.
- **docs/superpowers/plans/\*.md** = detailed task memory (already exists; keep as historical records, never delete).
- **docs/handover-YYYY-MM-DD.md** = historical handoffs (already exists; stay manual).

Two invocable Claude Code skills bookend each work cycle: one to open a session cheaply, one to save a concise handoff before `/clear` or `/compact`.

### Decisions (confirmed with user)
1. **Claude Code skills only** - no portable templates doc.
2. **Two skills** - `session-start` (opener) and `session-save` (handoff, with a mid-task mode).
3. **ACTIVE_CONTEXT.md only** - skills never auto-write dated handovers; those stay manual.
4. **Add a one-line CLAUDE.md pointer** so every session reads ACTIVE_CONTEXT.md even without the opener skill.

Anti-bloat is the whole point, so every artifact is length-capped and the skills forbid wholesale repo/plan inspection.

## Deliverable 1 - docs/ACTIVE_CONTEXT.md (new, short)

Fixed-structure file, capped at ~one screen (~45 lines). Seeded at implementation time from `git log` + `docs/handover-2026-06-05.md` to reflect true current state (most recent shipped work = Diagram Labelling activity, merged to main; before that = BackfillButton). Sections:

- `## Current project state` - 4-5 bullets, high level, no detail that CLAUDE.md already covers.
- `## Last completed task` - one short paragraph.
- `## Relevant recent files` - a few paths only.
- `## Current known issues` - bullets, or "none".
- `## Next recommended task` - the single next step.
- `## One plan to read next` - at most one `docs/superpowers/plans/*.md` path, if relevant.
- `## Session rules` - the short list the user specified (read CLAUDE.md + this file first; read only the one relevant plan; never inspect `docs/superpowers/plans` wholesale, `node_modules`, `.next`, `.swc`, `.superpowers`, `.playwright-mcp`, or screenshots unless directly needed; update this file before finishing).
- A `Last updated:` date stamp (absolute date).

## Deliverable 2 - .claude/skills/session-start/SKILL.md (new)

Frontmatter: `name: session-start`, `description: "Use at the start of a session to resume work or begin a specific task with minimal context - reads CLAUDE.md and docs/ACTIVE_CONTEXT.md only, summarizes state, scopes the relevant files, and proposes a small plan before editing."`

Body (the opener procedure):
1. Read ONLY `CLAUDE.md` and `docs/ACTIVE_CONTEXT.md`. Do not inspect the wider repo or any excluded dir (same exclusion list as the Session rules above).
2. Give a **5-bullet** summary of current state from ACTIVE_CONTEXT.md.
3. Set the goal: use the goal passed as the skill argument if present; otherwise use ACTIVE_CONTEXT.md's "Next recommended task". Avoid generic "continue where we left off".
4. Identify only the files/folders likely needed. Name at most one relevant plan under `docs/superpowers/plans/` and read it only if needed - never read all plans.
5. Propose a short implementation plan. If the change touches more than 2-3 files, wait for approval before editing.
6. Constraints block: minimise context; no unrelated inspection or refactors; do not touch auth, DB schema, or routing unless required; run only targeted tests/typechecks.
7. Closing reminder: run `/session-save` before `/clear` or `/compact`.

## Deliverable 3 - .claude/skills/session-save/SKILL.md (new)

Frontmatter: `name: session-save`, `description: "Use before /clear or /compact to write a concise handoff into docs/ACTIVE_CONTEXT.md so the next session resumes cleanly. Default mode = task complete (pre-clear); pass 'mid' for an in-flight checkpoint (pre-compact)."`

Body - two modes selected by the skill argument:
- **complete** (default, pre-`/clear`): rewrite ACTIVE_CONTEXT.md - Last completed task, Relevant recent files, commands/tests run + verification evidence, Current known issues, Next recommended task, and the one plan to read next (if any).
- **mid** (pre-`/compact`): rewrite for an in-flight handoff - current task goal, what's completed so far, files changed, files currently relevant, commands/tests already run, current failing errors/blockers, decisions made, and the exact next step after compact.

Shared rules in the body: keep it concise, no long implementation details, preserve the fixed section template, **overwrite stale content rather than appending** (and trim to ~45 lines if it grows), convert relative dates to absolute, and refresh the `Last updated:` stamp (get today's date via `Get-Date` if unknown).

## Deliverable 4 - CLAUDE.md (one small addition)

Add a short "Session workflow" note near the top (3 lines max, to honor "keep this file small"):
- At session start, read `docs/ACTIVE_CONTEXT.md` for current state (or run `/session-start`).
- Before `/clear` or `/compact`, run `/session-save` (`/session-save mid` for an in-flight checkpoint).

## Files to create / modify
- CREATE `docs/ACTIVE_CONTEXT.md`
- CREATE `.claude/skills/session-start/SKILL.md`
- CREATE `.claude/skills/session-save/SKILL.md`
- EDIT `CLAUDE.md` (add the 3-line Session workflow note)
- Untouched: all `docs/superpowers/plans/*`, `docs/superpowers/specs/*`, `docs/handover-*.md` (historical records, preserved per user).

## Verification
1. Confirm the three new files exist with valid YAML frontmatter and the agreed sections; `CLAUDE.md` edit is minimal and the file stays small.
2. ACTIVE_CONTEXT.md is under ~45 lines and accurately reflects current state (cross-check against `git log -5`).
3. Skill discovery: newly added project skills may require a session reload before they appear in the Skill tool list - note this to the user. Once available, `/session-start` should read only the two files and emit a 5-bullet summary; `/session-save` should rewrite ACTIVE_CONTEXT.md in place.
4. Dry run the loop conceptually: open with `/session-start [goal]`, finish, `/session-save`, then a hypothetical fresh session reads CLAUDE.md + ACTIVE_CONTEXT.md and can resume from "Next recommended task" without touching the wider repo.
