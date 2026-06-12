# Plan: Token Drain Remediation

## Context

The Codex adversarial review identified four concrete problems driving the 20-40% allowance burn on session resume:

1. **No hard quarantine** - `.gitignore` prevents git tracking but not model reads. Claude Code's broad `Read(Claude Code/**)` allow in `settings.local.json` lets both agents traverse `node_modules` (~500 MB) and `.next` (~1.1 GB) freely.
2. **Session-save cap is advisory** - The skill says "trim past ~45 lines" but `docs/ACTIVE_CONTEXT.md` is already 51 lines with stale Vercel URLs, smoke-test prose, and old PR details that reload on every resume.
3. **AGENTS.md instruction is self-contradicting** - Lines 1-4 require reading `node_modules/next/dist/docs/` before Next.js changes, which is impossible if `node_modules` is quarantined.
4. **Session-start has no size gate** - It reads `ACTIVE_CONTEXT.md` unconditionally regardless of how bloated it has grown.

No `.claudeignore`, `.agentignore`, or `.claude/settings.json` (project-level) exist today. Skills are duplicated identically in both `.agents/skills/` and `.claude/skills/` - both copies must be updated together.

---

## Changes

### 1. Create `.claude/settings.json` (new, committed)

Add project-level `permissions.deny` rules that hard-block reads of all build/artifact directories. These deny rules enforce quarantine regardless of the broad allow in `settings.local.json`.

```json
{
  "permissions": {
    "deny": [
      "Read(node_modules/**)",
      "Read(.next/**)",
      "Read(.swc/**)",
      "Read(out/**)",
      "Read(build/**)",
      "Read(dist/**)",
      "Read(coverage/**)",
      "Read(.turbo/**)",
      "Read(.vercel/**)",
      "Read(playwright-report/**)",
      "Read(test-results/**)",
      "Read(docs/screenshots/**)",
      "Read(.playwright-mcp/**)"
    ]
  }
}
```

### 2. Narrow `.claude/settings.local.json` (local-only, not committed)

Replace the over-broad allow:
```
"Read(//c/Users/jackw/OneDrive/Documents/Claude/Claude Code/**)"
```
with:
```
"Read(//c/Users/jackw/OneDrive/Documents/Claude/Claude Code/Study Tool Website/**)"
```
This removes cross-project spill while the deny rules in `settings.json` handle artifact quarantine within the repo.

### 3. Create `.claudeignore` (new, repo root, committed)

Signals the quarantine boundary to Claude Code's file traversal (and any future tooling that reads it):

```
node_modules/
.next/
.swc/
out/
build/
dist/
coverage/
.turbo/
.vercel/
playwright-report/
test-results/
docs/screenshots/
.playwright-mcp/
package-lock.json
*.tsbuildinfo
next-env.d.ts
diagram-audit-preview.png
```

### 4. Create `.agentignore` (new, repo root, committed)

Same list as `.claudeignore` for Codex companion traversal.

### 5. Create `docs/NEXTJS_16_NOTES.md` (new, committed, ~25 lines)

Extract the small set of Next.js 16 / Tailwind v4 facts that agents actually need. Content drawn from existing CLAUDE.md and AGENTS.md - no new information invented:

- `params` and `searchParams` are Promises; always `await` them
- App Router file conventions (layout.tsx, page.tsx, route.ts)
- Tailwind v4: `@import "tailwindcss"` + `@theme` in `src/app/globals.css`; do not create `tailwind.config.ts`
- Custom color tokens live in `@theme`
- API routes that may run long must export `maxDuration`

### 6. Update `AGENTS.md`

Replace the 4-line `<!-- BEGIN:nextjs-agent-rules -->` block (lines 1-4) that references `node_modules/next/dist/docs/` with a single line pointing to `docs/NEXTJS_16_NOTES.md`. Also update the "Next.js and code conventions" bullet that repeats the same `node_modules/next/dist/docs/` reference.

### 7. Rewrite session-save skill — hard cap

Update **both** copies:
- `.agents/skills/session-save/SKILL.md`
- `.claude/skills/session-save/SKILL.md`

Key changes:
- Lower cap from `~45 lines` to **35 lines hard maximum**
- Add explicit mandatory final step: after writing, count lines; if > 35, rewrite by dropping oldest detail until under cap — do not skip this step
- Ban: deployment URLs (unless they are the literal next action), historical PR prose, full smoke-test logs, stale operational facts
- Allow: ≤8 file paths, ≤3 verification bullets (pass/fail only, no output paste)

### 8. Rewrite session-start skill — size gate first

Update **both** copies:
- `.agents/skills/session-start/SKILL.md`
- `.claude/skills/session-start/SKILL.md`

Key changes:
- Make **Step 0** (before any read): check the line count of `docs/ACTIVE_CONTEXT.md`. If it exceeds 35 lines, do NOT read it. Instead: notify the user it is over budget and ask them to run `/session-save` to shrink it, or offer to shrink it inline before resuming.
- Cap the session-start output: 5 bullets maximum, 1 next command, ≤5 file paths.
- Add: "If `ACTIVE_CONTEXT.md` line count is unknown, read only the first line to check the `Last updated:` stamp and do a `(Get-Content ... | Measure-Object -Line).Lines` check before reading the body."

### 9. Trim `docs/ACTIVE_CONTEXT.md` immediately (local-only)

Reduce the current 51-line file to ≤35 lines by:
- Removing the full Vercel URL (stale preview deployment)
- Collapsing the 7-line verification section to 1 bullet: `npm test / lint / build: all passed (2026-06-07)`
- Removing "Current project state" Vercel CLI detail (not task-relevant)
- Keeping: branch, last task (2 sentences), 4-5 key recent files, known issues, next task

---

## Verification

1. After creating `.claude/settings.json` and `.claudeignore`: open a new Claude Code session and attempt `Read(node_modules/package.json)` - it should be blocked/denied without a prompt.
2. After narrowing `settings.local.json`: confirm reads of `src/` still work without extra prompts.
3. After skill updates: run `/session-save` and verify the output file is ≤35 lines. Run `/session-start` on a fresh session and verify it emits ≤5 bullets.
4. After AGENTS.md update: confirm `docs/NEXTJS_16_NOTES.md` covers the facts the old `node_modules/next/dist/docs/` reference was meant to provide.
5. Run `npm test`, `npm run lint`, `npm run build` - quarantine and skill changes touch no application code so these should pass unchanged.

---

## Files touched

| File | Action |
|------|--------|
| `.claude/settings.json` | Create (new) |
| `.claude/settings.local.json` | Narrow one allow entry |
| `.claudeignore` | Create (new) |
| `.agentignore` | Create (new) |
| `docs/NEXTJS_16_NOTES.md` | Create (new) |
| `AGENTS.md` | Update lines 1-4 + one bullet in conventions section |
| `.agents/skills/session-save/SKILL.md` | Rewrite cap rules + add final size check step |
| `.claude/skills/session-save/SKILL.md` | Same rewrite (duplicate) |
| `.agents/skills/session-start/SKILL.md` | Add Step 0 size gate |
| `.claude/skills/session-start/SKILL.md` | Same rewrite (duplicate) |
| `docs/ACTIVE_CONTEXT.md` | Trim to ≤35 lines (local-only, not committed) |
