# Redesign: Calm Premium Shell ("Tutor's Pen" identity)

## Context

The site reads corporate/cold/amateur despite a warm palette: mixed metaphors (Study Lab, Lab Bench, Research Shelves, specimens), a 768px-capped layout wasting desktop space, text-only click targets, native selects that render black on mobile, and no visible account identity. User wants an Apple-calm, Android-functional redesign. Brainstormed 2026-06-12; ten decisions resolved with user (visual companion used).

## Decisions (resolved with user)

- Identity overhaul, keep palette - parchment/sage/pastel tokens stay; structure, typography, polish change
- Naming: **Subject → Section → Materials** (maps 1:1 onto existing Project → Subfolder → Material; no schema change). Dashboard shelf groups demoted to optional collapsible Groups in Library; "Lab Bench" removed - ungrouped subjects list normally. Lab personality lives in microcopy only
- Desktop shell: **persistent left sidebar** (nav + subjects + profile at bottom). Mobile (<768px): **bottom tab bar** - Home / Library / Profile
- Today's Focus upgrade: **Continue where you left off** (recent session data exists) + **weak-spot suggestions** (mastery API exists)
- Typography: keep **Fraunces + Quicksand**, add **Caveat** as "tutor's pen" accent - personalised feedback, achievements, empty-state encouragement ONLY, never UI chrome
- Material manager: view in app (PDF/images native via signed URLs; docx = extracted text + download), download, rename
- Rename at every level (Subject/Section/Material) via kebab menu → dialog. Never inline/tap-to-edit, so no accidental mobile triggers; feature stays enabled at all screen sizes
- Motion: calm physics + signature moments. CSS-only, no new deps. Spring easing, staggered load fade-ups, session-complete tutor's-pen note, mastery bar fill
- Component fixes: whole-card clickable, custom styled dropdown (kill native select), uniform card sizing, centred metrics, wide canvas

## Roadmap supersession

This plan **supersedes** the UI phases of `docs/superpowers/specs/2026-06-10-ui-perf-cost-design.md` (P1 skeletons → folded into R1/R3; P4 pastel done; P6 micro-animations → R4; P7 dashboard calm-down + popover → R3/R4; P8 PWA stays light-done) and the remaining phases of `docs/superpowers/plans/2026-06-08-ui-modernisation.md` (PR2 mobile pass → R1; PR3 animations → R4; PR4 sound stays parked). Cost/perf phases (prompt caching, Haiku-first gating, activity migration P5) remain a separate track, untouched. Update both docs with a supersession note + pointer.

## Implementation phases (one branch/PR each, sequential)

### R1 - App shell (biggest structural change)
- New `src/components/layout/AppShell.tsx`: desktop grid `[sidebar 260px | main]`, replaces `PageShell.tsx` max-w-3xl cap; main content widens to ~max-w-6xl with responsive padding
- New `Sidebar.tsx`: brand, nav links (Home, Library), subject list (grouped), profile block at bottom (name/email + avatar initial + sign out)
- New `TabBar.tsx`: fixed bottom, 3 tabs, 44px+ targets, safe-area inset; sidebar hidden <md, tab bar hidden ≥md
- New routes: `src/app/library/page.tsx` (tree: Groups → Subjects → Sections), `src/app/profile/page.tsx` (account info, stats, sign out)
- `src/app/dashboard/page.tsx` becomes Home: keep content, adopt shell; remove breadcrumb Nav usage (Nav.tsx retired or reduced to mobile top bar with page title + avatar)
- Skeleton states for new shell surfaces (folds in old P1)

### R2 - Naming + rename everywhere
- UI copy sweep: Project→Subject, Subfolder→Section, shelves→Groups, remove Lab Bench/specimen/samples copy (DB/API names unchanged)
- Rename via existing `ui/Popover.tsx` kebab pattern → dialog for Subject, Section; extend to Groups
- Centralise nouns in a small constants file so future copy stays consistent

### R3 - Home utility + de-clutter
- Today's Focus panel: "Continue" card (last unfinished/most recent session → one-tap resume) + 2-3 weak-spot cards (lowest mastery via `src/app/api/projects/mastery/route.ts`)
- Metrics row: centred, reduced to what earns its place; Recent Sessions to right rail on desktop
- Exam prep selector: replace native `<select>` (dashboard page.tsx:467-475) with new `ui/Select.tsx` styled popover-listbox

### R4 - Component polish + tutor's pen + motion
- ProjectCard (`src/components/dashboard/ProjectCard.tsx:35-99`): whole card clickable (Link wraps card; action buttons stopPropagation), uniform min-height + line-clamp for name-length variance
- Popover/dropdown restyle: rounded-control corners, parchment surface, shadow-float, consistent on mobile
- Add Caveat via `next/font/google` in `src/app/layout.tsx`; token `--font-hand`; apply to coach notes, results encouragement, achievements, empty states
- Motion: staggered fade-up on Home load, spring transitions on tab/sidebar nav, session-complete pen-note animation, mastery-bar fill; respect existing `prefers-reduced-motion` block

### R5 - Material manager
- Section view lists materials: open (PDF/image inline via Supabase signed URL; docx/text → extracted text view + download), download, rename, delete
- New/extended API: signed-URL endpoint + material rename (PATCH); reuse storage paths from upload flow
- Viewer modal/route reusing `ui/Modal.tsx` patterns

## Verification (each phase)

- `npm test`, `npm run lint`, `npm run build` before merge
- Playwright/chrome-devtools visual pass at 375px, 768px, 1280px, 1536px: sidebar/tab bar swap, no horizontal scroll, tap targets ≥44px
- R3: confirm resume link lands in correct session; weak-spot cards match mastery API output
- R5: PDF + image + docx fixtures: view, download, rename round-trip
- Ship per CLAUDE.md workflow: branch → PR → Vercel green → merge

## Post-approval first steps

1. Write spec to `docs/superpowers/specs/2026-06-12-redesign-calm-premium-design.md`, commit; add supersession notes to the two old docs
2. Update memory (`project_ui_perf_cost_roadmap.md`) to point at new plan
3. Start R1 on a fresh branch off main
4. Stop visual companion server (mockups persist in `.superpowers/brainstorm/`)
