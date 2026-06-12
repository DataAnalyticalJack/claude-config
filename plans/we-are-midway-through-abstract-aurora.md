# Study Tool Optimisation — Generation Reliability, Crossword, Perf, Features

## Context

We are continuing optimisation of the study-tool app. Four needs surfaced:

1. **Case studies fail to generate for certain subjects.** Root cause found in
   [route.ts:185-191](src/app/api/generate/route.ts#L185-L191): the applied call
   (shortanswer + casestudy) is wrapped in `.catch(() => null)`, so any failure is
   silently swallowed and the study set saves with no applied content and no warning.
   Likely triggers: `max_tokens: 5000` is too tight for 8 short-answers + 3 multi-part
   case studies (truncation → malformed tool input), and Haiku struggles with the
   nested `casestudy` schema for some material.
2. **Crossword bugs**: no soft keyboard on mobile, only an all-or-nothing "Clear grid",
   and editing after a check wipes the ✓ state on every solved word.
3. **Performance/formatting** needs a deeper audit to cut latency and improve UX.
4. **Feature roadmap** — ship the queued regenerate-all-folders button and scope further ideas.

Confirmed decisions: Sonnet 4.6 for the **applied** call only (Haiku stays for core +
grading); crossword = per-word clear while keeping correct letters; sequence **generation
first, then crossword**; perf review is a **deeper audit** (after the two bug fixes).

---

## Phase 1 — Generation reliability + model routing (highest impact)

**File:** [src/app/api/generate/route.ts](src/app/api/generate/route.ts)

- Route the applied call to `claude-sonnet-4-6`; keep `claude-haiku-4-5-20251001` for the
  core call. The two are independent API requests, so prompt caching is unaffected.
- Raise applied `max_tokens` from 5000 to ~8000 to prevent truncation of case studies.
- Replace the silent `.catch(() => null)` with a helper that:
  - retries the applied call **once** on error or on `stop_reason === 'max_tokens'`;
  - on final failure returns `null` plus a captured reason (do not throw).
- Do **not** 500 the whole request when applied fails. Core always saves; when applied is
  missing, include a `warnings: string[]` field in the JSON response (e.g. "Short answers
  and case studies could not be generated for this material").
- Surface that warning gently in [folderId/page.tsx](src/app/projects/[id]/[folderId]/page.tsx)
  near the existing `error` state ([line 78, 102](src/app/projects/[id]/[folderId]/page.tsx#L78))
  — a non-blocking amber note, not an error. Reuse the existing `setError`/styling patterns
  rather than inventing a new component.

**Docs:** update the CLAUDE.md "AI model" line (10) to: Haiku for extraction, core
generation, and grading; Sonnet 4.6 for applied (short-answer + case study) generation.

**Verify:** generate a study set for a previously-failing subject; confirm `casestudy` and
`shortanswer` arrays are populated and the warning path works when forced. `npm run build`.

---

## Phase 2 — Crossword fixes

**File:** [src/components/crossword/CrosswordGame.tsx](src/components/crossword/CrosswordGame.tsx)

1. **Mobile keyboard** — add a single offscreen `<input>` (`inputMode="text"`,
   `autoCapitalize="characters"`, `autoCorrect="off"`, visually hidden, not `display:none`).
   Focus it in `selectCell` ([line 31](src/components/crossword/CrosswordGame.tsx#L31)) and in
   the clue-list buttons ([line 190](src/components/crossword/CrosswordGame.tsx#L190)) instead
   of `gridRef.current?.focus()`. Capture typed characters via the input's
   `onChange`/`onBeforeInput` and route through the existing letter logic in `handleKey`
   ([line 74](src/components/crossword/CrosswordGame.tsx#L74)); keep `onKeyDown` on the input
   so physical keyboards and arrows/Backspace still work on desktop. Clear the input value
   after each captured char so it stays empty.
2. **Stop the mark-wipe on edit** — `clearMarks` ([line 27](src/components/crossword/CrosswordGame.tsx#L27))
   currently does `setMarks({})` + `setChecked(false)`. Change letter/backspace handling so
   editing a cell clears **only that cell's** mark (and recomputes `won`), leaving every other
   word's ✓ intact. Keep `checked` true so `entrySolved` ([line 52](src/components/crossword/CrosswordGame.tsx#L52))
   still renders solved words.
3. **Per-word clear/retry** — for each **incorrect** entry after a check, add a small
   "Clear word" affordance in the clue list. It removes only that word's letters and marks,
   sets `sel` to the word's start, sets `dir`, and focuses the input. Correct words show ✓ and
   are left untouched. Keep the existing full "Clear grid" reset
   ([line 163](src/components/crossword/CrosswordGame.tsx#L163)).

**Verify:** Playwright on a real viewport — tap a cell on a mobile-sized session, confirm the
soft keyboard intent (focused input + typed letters land in cells); check answers, clear one
wrong word, confirm other words keep their state. Visually verify against color tokens.

---

## Phase 3 — Performance / formatting audit (deeper, after fixes)

Scope as an audit pass that produces a short findings list, then apply low-risk wins and
flag riskier refactors for approval. Target areas:

- **Client-vs-server components & fetch waterfalls.** The route pages are `'use client'` and
  fetch in `useEffect` (e.g. [folderId/page.tsx:80-85](src/app/projects/[id]/[folderId]/page.tsx#L80-L85)
  already parallelises with `Promise.all`; others may not). Look for sequential fetches that
  could parallelise or move server-side.
- **Loading states / Suspense** to reduce perceived latency.
- **Fonts and images** (next/font, next/image usage), and any layout shift.
- **List rendering**: stable keys, avoidable re-renders, memoisation on heavy lists.
- **Bundle**: large client components that could be server components.

Deliver findings inline (file:line + fix), apply the safe ones, list the rest for sign-off.

**Verify:** `npm run build` (bundle/type), and before/after compare on the heaviest route.

---

## Phase 4 — Feature roadmap

- **Regenerate all folders** (queued "next up"): a project-page button that fires one
  `/api/generate` call per subfolder so old cached sets gain short-answer + case-study
  content. Context in [projects/[id]/page.tsx](src/app/projects/[id]/page.tsx) (lists
  subfolders) and the generate call pattern in
  [folderId/page.tsx:93-104](src/app/projects/[id]/[folderId]/page.tsx#L93-L104). Needs
  per-folder progress and partial-failure handling. **Run `superpowers:brainstorming` before
  building** (sequential vs parallel calls, progress UI).
- **Further candidates to discuss** (not yet committed): spaced-repetition review scheduling,
  study streaks/progress, difficulty selector at generation time, export/share a study set.

---

## Notes

- `docs/ACTIVE_CONTEXT.md` and `docs/QUALITY_GATES.md` referenced in the prompt do **not**
  exist in the repo; this plan is grounded in CLAUDE.md, the memory index, and the code.
- Keep changes narrow per CLAUDE.md; no `tailwind.config.ts`; regular hyphens only; preserve
  the warm study-coach tone and existing color tokens.
