# Repo Review, Cleanup & Workflow Upgrade

## Context

Full review of the Study Tool Website repo requested: tidy redundant files, find optimization targets, prep for the UI/mobile push, harden the PR/deploy process, and improve Claude+Codex collaboration. Three-agent audit completed. Agreed scope this round: **cleanup + workflow tooling**; large code refactors are captured as a prioritized backlog for follow-up branches, not executed now.

Key audit facts:
- `.agents/skills/` is a deliberate Codex mirror of `.claude/skills/` (commit 8ef7d78: "all skills mirrored in .agents/skills for Codex") - keep it, but it has zero drift protection.
- `docs/superpowers/` is in `.gitignore` (line 41) but 11 files (~250KB) are still tracked - rule never took effect.
- 5 branches are fully merged (verified via `git cherry` - all show `-`): local `codex/pr-1-1-validation-baseline`, `diagram-labelling`, `feat/diagram-quality-batch1`; remote `origin/codex/pr-1-1-validation-baseline`, `origin/feat/diagram-quality-batch1`, `origin/feat/p4-pastel-fraunces`, `origin/feat/p8-pwa-a11y`.
- No CI exists; validation is fully manual. No `typecheck` script.
- `docs/CODEX_AUDIT.md` and `docs/IMPLEMENTATION_ROADMAP.md` last touched 2026-06-06; their claims (failing tests) contradict the current green state (180 tests passing 2026-06-11).
- Known lint error: `src/components/dashboard/DraggableDashboard.tsx:102`. 6 `console.log` calls in `src/app/api/generate/route.ts` (lines ~191, 262, 266, 269, 273, 309).

## Phase 1: Repo cleanup (low-risk, direct to main per CLAUDE.md rule)

1. **Untrack superpowers docs**: `git rm -r --cached docs/superpowers` - files stay on disk, gitignore intent finally honored. Repo loses ~250KB of shipped plan docs.
2. **Prune merged branches**: delete the 3 merged local branches; delete remote `codex/pr-1-1-validation-baseline`, `feat/diagram-quality-batch1`, `feat/p4-pastel-fraunces`, `feat/p8-pwa-a11y`; `git fetch --prune`.
3. **Skill mirror drift guard**: add `scripts/sync-skills.mjs` (copy `.claude/skills/*` -> `.agents/skills/*`) + `npm run sync:skills`; add one line to `docs/AI_WORKFLOW.md`: run after any skill edit. (Currently identical - verified by diff - so first run is a no-op.)
4. **Refresh stale docs**: update `docs/CODEX_AUDIT.md` header to mark resolved findings resolved (or move to `docs/archive/`); refresh `docs/IMPLEMENTATION_ROADMAP.md` status to current reality (auth P1-P4 shipped, tests green).

## Phase 2: Workflow tooling

1. **CI gate** (new `.github/workflows/ci.yml`): on `pull_request` and `push` to `main` - `npm ci`, `npm run lint`, `npm test`, `npm run build`.
   - Build needs Supabase/Anthropic env vars; use harmless placeholders in workflow `env:` (e.g. `NEXT_PUBLIC_SUPABASE_URL: https://placeholder.supabase.co`). Verify locally first that `npm run build` succeeds with placeholders; if a module-scope client throws, fall back to GitHub repo secrets (user adds them).
2. **`typecheck` script**: add `"typecheck": "tsc --noEmit"` to `package.json` for fast feedback without a full build.
3. **Code hygiene** (touches app code -> feature branch `chore/code-hygiene` + PR):
   - Fix the lint error at `DraggableDashboard.tsx:102`.
   - Remove or gate the 6 `console.log` calls in `src/app/api/generate/route.ts` behind `process.env.NODE_ENV !== 'production'` or a `DEBUG_GENERATE` flag (keep token-usage visibility - it is cost telemetry; recommend gating, not deleting).
4. Note CI in `docs/AI_WORKFLOW.md` ship checklist ("wait for CI green + Vercel preview").

## Phase 3: Write the optimization backlog into docs/IMPLEMENTATION_ROADMAP.md

Refresh the roadmap with this prioritized backlog (each item = its own future branch/PR), aligned with the approved nine-phase UI/perf/cost roadmap:

1. **Mobile responsive pass** (highest value for stated goal): only 13 responsive-prefix usages across 40 components. Fix `grid-cols-2` without `sm:` fallback in `DraggableDashboard.tsx:167`, fixed `max-w-[160px]` buttons in `TrueFalseActivity.tsx:37`, inline `height: 220` in `FlashcardActivity.tsx`, 32px IconButton touch targets, missing truncate/overflow on long names. Establish Playwright 375px-viewport screenshot as the standard UI verification step.
2. **Activity component dedup**: extract a shared reveal-cycle wrapper (`select -> reveal -> explanation -> next`) used by all 8 activity components in `src/components/study/activities/` (~300-400 duplicated lines).
3. **Code-split activities**: `next/dynamic` per activity type in `src/components/study/SessionRunner.tsx` (~40KB less JS per session).
4. **Externalize `src/lib/diagrams/templates.ts`** (888 lines of inline SVG) to `/public/diagrams/` JSON loaded on demand.
5. **API route helper**: shared auth-check + error-response wrapper across the 18 CRUD routes; plus `getNextSortOrder()` helper for the 3 count-then-insert routes.
6. **Dashboard LCP**: `src/app/dashboard/page.tsx` (550 lines, all-client, `ssr:false` dynamic import) - add Suspense boundaries / server-component sections.
7. **Test gaps**: `SubfolderList.tsx`, `CrosswordGame.tsx` UI, most activity components untested.

## Phase 4: Final report to user (chat, not files)

Deliver in the closing message:
- **Workflow/automation suggestions**: which existing skills are healthy (all 5 are); candidates to add - a `mobile-check` skill (Playwright 375px screenshot routine), a PostToolUse hook idea for auto-lint on edit, `/fewer-permission-prompts` run, statusline. Recommend NOT adding husky/prettier (user declined pre-commit; AI workflow already validates).
- **Claude+Codex synergy**: current state (shared AGENTS.md, ACTIVE_CONTEXT.md, mirrored skills, branch separation rules) + improvements: skill-sync script (done in Phase 1), CI as the neutral arbiter both agents must pass, roadmap doc as single source of truth for who-does-what, Codex for second-opinion review on Claude PRs via `codex:rescue`.
- **Overlooked items**: custom domain (ACTIVE_CONTEXT flags `study-tool-xi.vercel.app` as ad-hoc), Supabase Storage orphan cleanup for deleted materials, rate limiting on `/api/generate` (now that auth exists, per-user limits possible), error monitoring (Sentry or Vercel logs review), DB index review against RLS owner policies (Supabase advisors), E2E smoke test for the auth + study-session happy path.

## Files to touch

- New: `.github/workflows/ci.yml`, `scripts/sync-skills.mjs`
- Edit: `package.json` (typecheck, sync:skills), `docs/AI_WORKFLOW.md`, `docs/IMPLEMENTATION_ROADMAP.md`, `docs/CODEX_AUDIT.md`
- Branch PR only: `src/app/api/generate/route.ts`, `src/components/dashboard/DraggableDashboard.tsx`
- Git ops: `git rm -r --cached docs/superpowers`, branch deletions, `git fetch --prune`

## Verification

- Phase 1: `git status` clean after commit; `git ls-files docs/superpowers/` returns empty; `git branch -a` shows only `main` + active branches; `diff -r .claude/skills .agents/skills` empty after sync script run.
- Phase 2: `npm run typecheck` passes; `npm run build` with placeholder env vars locally before pushing CI config; CI run green on the `chore/code-hygiene` PR (this PR doubles as the CI smoke test); `npm run lint` exits 0 after the DraggableDashboard fix; `npm test` all green.
- Ship per AI_WORKFLOW.md: hygiene PR merged after CI + Vercel green; report deployment URL; leave checkout on clean main.
