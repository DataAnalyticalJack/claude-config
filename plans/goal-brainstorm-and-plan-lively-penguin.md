# Plan: 80/20 Frontend UI/UX Modernisation

## Context

The Study Tool Website works but reads as an "Android/school-project" UI: every button, card,
and modal is hand-rolled inline (there is **no** shared `Button`/`Card`/`Modal`), touch targets dip
below 44px, the flashcard has a fixed `height: 220px`, some actions hide behind `group-hover`
(invisible on touch), and there is **no** sound layer. The goal is a fluid, warm, modern web app
that doubles as a clean precursor to a native mobile port.

Good news from exploration — the base is healthier than the brief assumes, so this is a true 80/20:
- A coherent token system already exists in `src/app/globals.css` (`@theme` with sage/parchment/sand,
  `--font-sans`, `popIn`/`fadeUp`/`calmShift` keyframes, and 3D-flip utilities).
- A correct `@media (prefers-reduced-motion: reduce)` block already exists (lines ~76-89) — we extend it.
- The Matching activity is **already click-based, not drag-and-drop** — the brief's "pointer-only drag"
  porting risk is already solved. The only true hover/pointer gap is the grip in `SubfolderList.tsx`.

**Locked decisions:** warm **clay** accent layered on the existing sage base; add **lucide-react**;
sound system ships **default-muted** with a persisted toggle and placeholder assets.

The work is sequenced into **4 PR-sized steps**, each independently shippable and verifiable.

---

## PR 1 — Design-system foundation (tokens + primitives + icons)

**Goal:** one source of truth for color/radius/shadow/motion, plus shared primitives that the rest of
the app migrates onto. Highest-leverage PR.

### 1a. `src/app/globals.css` — extend the `@theme` block

Add (Tailwind v4 auto-generates the utilities shown in comments):

```css
@theme {
  /* --- warm accent (new) --- */
  --color-clay-50:  #fbf0ea;
  --color-clay-100: #f3d8c9;
  --color-clay-300: #e0a487;
  --color-clay:     #C97B5A;   /* bg-clay / text-clay  -> primary CTA + highlight */
  --color-clay-600: #b1654a;   /* hover */
  --color-honey:    #D8A24A;   /* warn/streak, replaces raw amber where wanted */

  /* --- warmed neutrals (override existing values) --- */
  --color-parchment: #FAF6EF;  /* was #FDFCF7 — slightly warmer page base */
  --color-surface:   #FFFDF8;  /* new: card/surface tone, replaces bare bg-white */
  --color-sand:      #E7DECB;  /* was #E5DFD0 — warmer borders */

  /* --- soft shadow tokens (new) -> shadow-soft / shadow-float / shadow-pressed --- */
  --shadow-soft:    0 2px 8px -2px rgb(45 58 45 / 0.08), 0 1px 3px rgb(45 58 45 / 0.06);
  --shadow-float:   0 8px 24px -6px rgb(45 58 45 / 0.12), 0 2px 6px rgb(45 58 45 / 0.08);
  --shadow-pressed: inset 0 1px 2px rgb(45 58 45 / 0.10);

  /* --- standardised radius semantics (new) -> rounded-card / rounded-control --- */
  --radius-control: 0.875rem;  /* buttons, inputs (between lg and xl) */
  --radius-card:    1rem;      /* cards (xl) */
  --radius-sheet:   1.5rem;    /* modals/large surfaces (2xl) */

  /* --- motion tokens (new) -> ease-soft / ease-spring + animations --- */
  --ease-soft:   cubic-bezier(0.22, 0.61, 0.36, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);

  --animate-modal-in: modalIn 0.22s var(--ease-soft) both;   /* PR3 uses these */
  --animate-shake:    shake 0.32s ease-in-out both;
  --animate-celebrate: celebrate 0.5s var(--ease-spring) both;
}
```
Keyframes (`modalIn` = opacity+scale, `shake` = translateX, `celebrate` = scale pop) are added in
the cascade layer below `@theme`; all use only `transform`/`opacity`. Extend the existing
`prefers-reduced-motion` block to also neutralise the new animations (it already wildcards `*`, so
mainly verify the new names degrade to static).

**Convention going forward:** `rounded-card` for cards, `rounded-control` for buttons/inputs,
`rounded-sheet` for modals; `shadow-soft` default, `shadow-float` on elevation/hover. No raw
`bg-white` — use `bg-surface`.

### 1b. Add lucide-react

`npm i lucide-react` (tree-shakeable; warm rounded line set fits the aesthetic).

### 1c. New shared primitives — `src/components/ui/`

| File | Responsibility |
|------|----------------|
| `Button.tsx` | Variants `primary` (clay), `secondary` (sage), `outline`, `ghost`; sizes `sm`/`md`/`lg`; optional `icon` (lucide) + `iconPosition`; built-in `min-h-[44px]`, `rounded-control`, `active:scale-[0.97]`, `focus-visible:ring-2 focus-visible:ring-clay/sage`, `disabled` + `loading` (spinner) states. |
| `IconButton.tsx` | Square ≥44px tap target, aria-label required, same press/focus tokens. Replaces tiny text-only controls. |
| `Card.tsx` | `bg-surface border border-sand rounded-card shadow-soft`, optional `interactive` (hover `shadow-float`, `active:scale-[0.99]`). |
| `Modal.tsx` | Portal + backdrop, `rounded-sheet`, `animate-modal-in`, Escape-to-close, focus trap + restore, `role="dialog"` `aria-modal`. Extracts the inline pattern from `NewProjectModal`. |

### 1d. Proof migrations (keep PR reviewable)

Migrate two existing surfaces onto the primitives to validate the API:
`src/components/dashboard/NewProjectModal.tsx` (-> `Modal` + `Button`) and
`src/components/layout/Nav.tsx` home control (-> `IconButton` + lucide `Home`).

**Verify:** `npm run build` + `npm run lint`; visually confirm dashboard + new-project modal unchanged
in layout but warmer/softer; confirm focus-visible rings appear on keyboard tab.

---

## PR 2 — Mobile-first interaction pass

**Goal:** migrate the high-traffic interactive surfaces onto the primitives and close the touch/a11y gaps.

- **Activities -> `Button`/`IconButton`:** `MCQActivity`, `FlashcardActivity`, `TrueFalseActivity`,
  `FillInBlankActivity`, `MatchingActivity`, `ShortAnswerActivity`, `CaseStudyActivity`
  (`src/components/study/activities/`). Removes ~8 sets of hand-rolled button classes.
- **Add `focus-visible` + 44px** to option buttons in `MCQActivity` and `MatchingActivity` (currently
  rely on browser default focus and can be short).
- **Flashcard responsive height:** replace inline `height: 220` with `min-h-[12rem] h-auto` (+ `aspect`
  guard) so long content/landscape no longer overflow. Keep the existing 3D flip structure.
- **Thumb-friendly session actions:** in `src/components/study/SessionRunner.tsx`, move the
  Prev/Skip/Next row into a sticky bottom action bar on mobile
  (`sticky bottom-0 ... pb-[env(safe-area-inset-bottom)]`, reverts to inline at `sm:`), so the primary
  action sits in the thumb zone. Stack buttons full-width on the narrowest screens.
- **Kill hover-only reveal:** `src/components/project/SubfolderList.tsx` grip — drop
  `opacity-0 group-hover:opacity-100` / `touch-none` so the handle is always reachable on touch
  (keep a subtle desktop de-emphasis via opacity, not full hide).
- **StreakBadge placement:** `src/components/study/StreakBadge.tsx` `fixed top-16` -> avoid overlap with
  the new sticky bar / add safe-area awareness.

**Verify:** `npx jest --testPathPatterns=RecentSessions` (and any activity tests), `npm run lint`,
`npm run build`; DevTools device emulation — every interactive target ≥44px, no action hidden without
hover, primary session button reachable one-handed.

---

## PR 3 — Micro-animations & fluidity

**Goal:** 4 high-impact, GPU-cheap motion moments (transform/opacity only, 60fps).

1. **Flashcard flip** — keep the CSS 3D transform; standardise timing to `--ease-soft` and ensure the
   front/back swap is crisp (`FlashcardActivity`).
2. **Modal reveal** — `Modal` uses `animate-modal-in` (backdrop fade + sheet scale-up) with an exit
   transition.
3. **Answer feedback** — correct answer: `animate-celebrate` pop on the chosen option / score; wrong
   answer: `animate-shake` on the option. Wire in `MCQ`, `TrueFalse`, `Matching`, `ShortAnswer`.
4. **Session-complete success** — `src/components/results/ScoreHero.tsx`: staged `fade-up` +
   `celebrate` on the score reveal.

All new motion uses the PR1 keyframes (pure `transform`/`opacity`). Confirm the extended
`prefers-reduced-motion` block degrades every one to a static end-state.

**Verify:** `npm run build`; toggle OS "reduce motion" and confirm animations flatten; visually confirm
no jank (no layout-affecting properties animated).

---

## PR 4 — Sound design layer (default-muted)

**Goal:** lightweight, non-blocking audio feedback with a global mute toggle.

- **`src/lib/sound/SoundProvider.tsx`** (client) — context holding `enabled` (default **false**,
  persisted to `localStorage`), lazy-creates a single `AudioContext`/`HTMLAudio` pool, unlocks on first
  user gesture (autoplay-policy safe), no-ops when muted or reduced-motion-preferring users opt in.
- **`useSound()` hook** — `play('select' | 'correct' | 'error' | 'complete')`, fire-and-forget,
  swallows errors so audio never blocks UI.
- **`src/lib/sound/sounds.ts`** — maps names to asset paths.
- **Assets (placeholders to drop final files into):** `public/sounds/select.mp3` (soft click),
  `correct.mp3` (warm chime), `complete.mp3` (success swell), `error.mp3` (soft thud).
- **Toggle:** `src/components/ui/SoundToggle.tsx` (lucide `Volume2`/`VolumeX`) mounted in
  `src/components/layout/Nav.tsx`.
- **Mount provider** in `src/app/layout.tsx` wrapping `children`.
- **Wire calls:** `select` on option pick (MCQ/Matching/TrueFalse), `correct`/`error` on answer reveal,
  `complete` on session finish (`SessionRunner` / `ScoreHero`).

**Verify:** `npm run build` + `npm run lint`; manually toggle sound on, confirm each event plays once and
nothing plays while muted; confirm default state is muted on a fresh profile; confirm no audio errors in
console when assets are placeholder/empty.

---

## Cross-cutting verification (before any merge)

Per `CLAUDE.md`: focused tests + `npm test` + `npm run lint` + `npm run build` must pass. Each PR goes
on its own feature branch off `main`, reviewed, then shipped to `main` with Vercel success confirmed.
Visually compare against current screens to confirm warmth/softness improved without layout regressions.

## Files at a glance

- **Tokens/motion:** `src/app/globals.css`
- **New primitives:** `src/components/ui/{Button,IconButton,Card,Modal,SoundToggle}.tsx`
- **Sound:** `src/lib/sound/{SoundProvider.tsx,sounds.ts}`, `public/sounds/*`, `src/app/layout.tsx`
- **Migrations:** `src/components/study/activities/*`, `src/components/study/SessionRunner.tsx`,
  `src/components/results/ScoreHero.tsx`, `src/components/dashboard/NewProjectModal.tsx`,
  `src/components/layout/Nav.tsx`, `src/components/project/SubfolderList.tsx`,
  `src/components/study/StreakBadge.tsx`
- **Dependency:** `lucide-react`
