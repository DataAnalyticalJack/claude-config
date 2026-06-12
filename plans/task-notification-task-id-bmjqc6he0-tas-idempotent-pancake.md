# Session plan: spec commit + R1 app shell

## Context

The calm-premium redesign ("tutor's pen" identity) was brainstormed and fully resolved with the user on 2026-06-12; the roadmap lives only in a local plan file (`C:\Users\jackw\.claude\plans\lite-brainstorming-grill-me-frontend-des-precious-karp.md`). This session lands it in-repo and implements Phase R1, the app shell: persistent desktop sidebar, mobile bottom tab bar, new Library and Profile routes, and a wide canvas replacing the 768px cap that makes the site feel cramped on desktop.

## Part A - docs commit (low-risk, direct to main per CLAUDE.md)

1. Copy the roadmap into `docs/superpowers/specs/2026-06-12-redesign-calm-premium-design.md`.
2. Add supersession notes pointing at the new spec:
   - `docs/superpowers/specs/2026-06-10-ui-perf-cost-design.md` (UI phases P1, P4, P6, P7, P8 superseded; cost/perf phases stay live).
   - `docs/superpowers/plans/2026-06-08-ui-modernisation.md` (PR2, PR3 superseded; PR4 sound stays parked).
3. Update memory `project_ui_perf_cost_roadmap.md` to point at the new spec; add a redesign memory if useful.
4. Remove the now-consumed `Handoff doc:` line from local `docs/ACTIVE_CONTEXT.md` (not committed).
5. Commit + push directly to `main`, report hash.
6. Stop the visual companion server if still running (port 64983).

## Part B - R1 app shell (branch `feat/redesign-r1-app-shell`)

New components in `src/components/layout/`:

- **`AppShell.tsx`** - replaces PageShell internals. Desktop (`md:`): grid `[260px sidebar | main]`; main content `max-w-6xl` with responsive padding. Mobile: no sidebar, bottom padding for tab bar, slim top bar (page title + avatar initial).
- **`Sidebar.tsx`** (client) - brand, nav links (Home, Library), subject list (fetched via existing `@/lib/supabase/client`, grouped like dashboard shelves), profile block at bottom (name/email, avatar initial, sign out - reuse the `handleSignOut` pattern from `Nav.tsx`). Hidden below `md`.
- **`TabBar.tsx`** (client) - fixed bottom, 3 tabs (Home, Library, Profile), 44px+ targets, `env(safe-area-inset-bottom)`, active state from `usePathname`. Hidden at `md` and up.

Adaptation strategy (keeps the diff small):

- Keep `PageShell.tsx`'s `{ children, breadcrumbs }` API and rewrite its body to render `AppShell`, so the 14 existing call sites need no edits in R1. Breadcrumbs prop feeds the mobile top-bar title (last crumb) for now; full breadcrumb trail retires.
- `Nav.tsx` is no longer rendered (replaced by sidebar/top bar/tab bar); delete it once nothing imports it.

New routes:

- `src/app/library/page.tsx` - tree view: Groups -> Subjects -> Sections, links into existing project/folder pages. Plus `loading.tsx` skeleton.
- `src/app/profile/page.tsx` - account identity at a glance (email/name from Supabase session), basic stats, sign out. Plus `loading.tsx` skeleton.

`src/app/dashboard/page.tsx` becomes Home: keep content, adopt the shell, widen to the new canvas. Content reflow/de-clutter waits for R3; only shell adoption here.

Skeleton states for sidebar subject list and the two new routes (folds in old P1).

## Out of scope this session

R2-R5 (naming sweep, Home utility, polish/motion/Caveat, material manager), the native select replacement, and the two known diagram issues.

## Verification

- `npm test`, `npm run lint`, `npm run build` all pass.
- Visual pass via browser tools at 375px, 768px, 1280px, 1536px: sidebar/tab bar swap at the `md` boundary, no horizontal scroll, tap targets >= 44px, sign out works from sidebar and profile.
- Spot-check 3 existing pages (dashboard, project, folder) render correctly inside the new shell.
- Ship per AI_WORKFLOW: PR -> checks green -> Vercel green -> merge -> report URL.
