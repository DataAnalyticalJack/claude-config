# Study Tool Website: UI/UX, Mobile, Performance & AI-Cost Plan

## Context

Planning pass only (no code this session). Goal: move the app toward Apple-style calm with a wider pastel palette, make mobile/tablet feel fluid and app-port-ready, cut real and perceived load times, and reduce Anthropic generation cost - all without rebuilding the existing upload -> generate -> study -> results framework.

Decisions locked with user:
- **Design**: pastel evolve, incremental (keep parchment/sage base, widen accents; no repaint)
- **AI cost**: Anthropic-only tuning (no second provider)
- **Old ui-modernisation plan**: fold PRs 2-3 (activity primitives, micro-animations) into this roadmap; sound layer parked
- **Mobile**: responsive + light PWA (manifest + icons, NO service worker)
- **Display font**: Fraunces via `next/font/google` (headings only; body stays system stack)
- **P9 gate**: env flag + logged comparison before default-on
- **PWA icon**: generate simple sage-on-parchment monogram

## Executive summary

Foundations are healthy: token system, four UI primitives, a11y baseline, and reduced-motion support shipped in PR1 of the prior ui-modernisation plan. The gaps are: activity screens not on primitives, zero `loading.tsx` files (blank server-route transitions), client-fetch dashboard, an unscoped mastery query, an oversized sessions payload, and a generate route that pays full input price 3x for the same 80K-char material because parallel firing defeats its own prompt cache. Nine small PRs fix these in value order; quick wins (skeletons, cache restructure, query slimming) land first.

## Current UX/design diagnosis

- Tokens: sage/clay/honey/parchment/sand, 3 shadow levels, 3 radii, 6 keyframes, reduced-motion handled. System fonts only; no dark mode.
- Primitives (`src/components/ui/`): Button (44px min), IconButton, Card, Modal. Only NewProjectModal + Nav use them; 8 activity components are hand-rolled Tailwind.
- Clutter flags: dashboard mixes 5+ card types; folder edit row is 4 outline buttons; 28px colour swatches; drag grips `opacity-0` on hover = invisible on touch; tight mobile hero.
- A11y: focus-visible rings widespread; Modal focus trap untested; no arrow-key support on swatch groups.

## Recommended design direction

**Pastel evolve, incremental.** Keep warm parchment/sage base. Add pastel accent pairs (mint/sky/lilac/peach/butter - each a bg + ink shade meeting 4.5:1) in `@theme`, mapped to subjects and activity types via a small lookup (`src/lib/theme.ts`). Fraunces for headings only. Calm comes from hierarchy + spacing, not decoration: fewer card types, popovers over button rows, generous whitespace, motion reserved for feedback moments (existing celebrate/shake/fadeUp keyframes). Avoids gradients-on-white AI slop; personality carried by type + targeted pastel accents.

## Mobile/tablet improvements

- Sticky bottom action bar in SessionRunner on small screens (thumb-reach Prev/Skip/Next)
- Drag grips always visible under `@media (pointer: coarse)`
- Colour swatches to 44px with arrow-key roving focus
- Flashcard responsive height (no fixed 220px)
- Modals become bottom-sheet style on small screens (CSS only; `rounded-sheet` token exists)
- Nav mobile spacing polish
- Light PWA: `src/app/manifest.ts` + 192/512/maskable + apple-touch icons, parchment theme-color -> installs to home screen, eases future app port

## Performance and perceived speed

- `loading.tsx` skeletons for all 5 server-component routes (study, results, history, crossword, study-all) - currently blank during fetch
- Mastery API: scope to dashboard's project ids or aggregate in SQL - today it joins ALL sessions across ALL projects
- `/api/sessions/[subFolderId]`: explicit columns + limit, stop shipping all nested answers when unused
- study-all: collapse N+2 queries to 2 (already parallel; the cost is query count)
- SessionRunner: add `useMemo`/`useCallback` on hot paths while migrating it in P5
- Dynamic-import dnd-kit on dashboard (`next/dynamic`) - only drag consumer
- Dashboard stays client-component (conversion = high-risk rewrite; skeletons + P3 slimming capture most of the win)

## AI cost reduction (Anthropic-only)

Verified facts: 3 parallel calls per generation (Haiku core 6000 max_tokens / Sonnet applied 8000, retry-once / Haiku diagram 500); 80K-char material (~20K tokens) in each; `cache_control: ephemeral` present but parallel `Promise.all` firing means each call pays full input. Caches are model-scoped AND prefix-exact, so the fix needs restructuring, not reordering alone.

**P2 mechanism (safe, ~8-10% off total generation cost, ~45% off Haiku input):**
1. Unify Haiku prefix: both Haiku calls get identical `tools` array and identical `system` (instruction + material block with `cache_control`); diagram registry summary moves into the diagram call's user message
2. Fire diagram call (cheap) + Sonnet applied simultaneously; await diagram (~1-3s), then fire core as a cache read (0.1x on ~20K tokens). Sonnet is the latency long pole, so wall time unhurt
3. Keep `cache_control` on Sonnet's material block: its retry path reads at 0.1x instead of re-paying
4. Cache miss = today's price exactly; log `cache_read_input_tokens` for visibility. Sub-4096-token materials silently don't cache (Haiku minimum; harmless)

**P9 mechanism (gated, ~35-45% off when it lands):** Haiku-first applied generation sharing P2's cached prefix, strict schema validation, Sonnet fallback on failure/truncation. Behind env flag; log fallback rate + warnings across ~10 real generations before default-on.

**Considered and dropped:** retry reduction (already retry-once - verified `attempt < 2` at `src/app/api/generate/route.ts:217`); exam-prep text caching (already reused from `materials`; resend cost handled by P2); material trimming (no relevance signal, quality risk); batched grading (~$0.0005/call, negligible); second provider / offline PWA / sound / dark mode (ruled out by locked decisions).

## PR-sized roadmap

Baseline validation for every phase: `npm test`, `npm run lint`, `npm run build`.

| # | Phase | Effort | Risk | Depends on |
|---|-------|--------|------|------------|
| P1 | Route loading skeletons | S | low | - |
| P2 | Cache-friendly /api/generate restructure | M | med | - |
| P3 | Query/payload slimming | M | low | - |
| P4 | Pastel tokens + Fraunces | S | low | - |
| P5 | Activity primitives migration + mobile session UX | L | med | P4 |
| P6 | Micro-animations | S | low | P5 |
| P7 | Dashboard calm-down + popover + nav polish | M | med | P4 |
| P8 | Light PWA + a11y hardening | S | low | - |
| P9 | Haiku-first applied generation (env-flagged) | M | med-high | P2 |

**P1** - new `loading.tsx` under `src/app/projects/[id]/[folderId]/{study,results,history,crossword}/` and `src/app/projects/[id]/study-all/`, parchment-toned skeletons.
**P2** - `src/app/api/generate/route.ts` per mechanism above; optional usage-logging helper.
**P3** - `src/app/api/projects/mastery/route.ts`, `src/app/api/sessions/[subFolderId]/route.ts`, `src/app/projects/[id]/study-all/page.tsx`, `src/app/page.tsx` (caller).
**P4** - `src/app/globals.css` (@theme pastel pairs, `--font-display`), `src/app/layout.tsx` (next/font), new `src/lib/theme.ts` (subject/activity accent map).
**P5** - 8 files in `src/components/study/activities/`, `src/components/study/SessionRunner.tsx` (sticky bar + memoization), `src/components/project/SubfolderList.tsx` (grip fix), `src/components/ui/Button.tsx` (variants if needed).
**P6** - FlashcardActivity flip, `src/components/ui/Modal.tsx` reveal, answer feedback celebrate/shake wiring, ScoreHero sequence, flip keyframe in globals.css.
**P7** - `src/app/page.tsx`, ProjectCard, SubfolderList, `src/components/layout/Nav.tsx`, new `src/components/ui/Popover.tsx` (headless, no new dep), dnd-kit via `next/dynamic`.
**P8** - new `src/app/manifest.ts`, `public/icons/*` (sage monogram), layout metadata, new `src/__tests__/modal.test.tsx` (focus trap).
**P9** - `src/app/api/generate/route.ts`, env flag, fallback-rate logging.

## Validation plan per phase

- **P1**: devtools network throttle, navigate all 5 routes - skeleton then content, no layout shift
- **P2**: dev-log `usage.cache_read_input_tokens` > 0 on core call for a >=16K-char material; output schema diff vs current; delta/refresh modes still work; wall time within +-10%
- **P3**: mastery percentages identical before/after on real data; network-tab payload sizes compared; history renders unchanged
- **P4**: build passes (font subset at build time); heading sweep; pastel pairs contrast-checked 4.5:1
- **P5**: activity tests pass; manual run-through of every activity type at 375px; 44px targets; no double-submit from sticky bar
- **P6**: reduced-motion toggle disables all new animations; nothing animates on initial paint
- **P7**: drag-reorder works after dynamic import; `next build` bundle diff; popover keyboard nav (Esc/Tab/arrows)
- **P8**: Lighthouse installable check; real Android/iOS add-to-homescreen; focus-trap tests pass
- **P9**: env-flag off = byte-identical behavior; flag on = fallback rate + warnings logged across ~10 generations, user reviews before default-on

## Open questions

None blocking - all resolved with user (direction, provider, old-plan fold-in, PWA scope, font, P9 gate, icon).

## Post-approval notes

- Implementation starts from a fresh feature branch per phase, per AGENTS.md / docs/AI_WORKFLOW.md
- Brainstorming skill's spec step: this plan doubles as the spec; if a separate spec doc is wanted, write to `docs/superpowers/specs/2026-06-10-ui-perf-cost-design.md` after approval
- Uncommitted RLS migrations + CLAUDE.md edits are still staged from the prior session - commit those first (separate concern, low-risk path) before starting P1
