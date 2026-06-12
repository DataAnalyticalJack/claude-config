# Fix bespoke diagram labelling: kill baked-in text + accurate, uncramped pins

## Context

The bespoke Gemini diagram pipeline (`src/lib/gemini/*`) generates an unlabelled
illustration, then asks Gemini to point at each structure so the app can overlay
numbered pins. Two failures persist (gold standard = the bronchial-tree image):

1. **Text leaks into the image.** `gemini-2.5-flash-image` bakes labels/letters in
   despite a strong "no text" prompt. Prompt instructions alone are unreliable.
2. **Pins are cramped / misaligned / wrong.** Gemini self-spaces the numbered
   bubbles ("anchors") poorly, so they cluster (see the alveolus screenshot), and
   when validation fails the whole diagram is discarded (`null`).

Codex already landed sound groundwork: a `tip`/`anchor` split (bubble in empty
space + leader line to the structure), separate tip/anchor distance validation,
de-slugified `correctTerm` in the regenerate route, and an optional
`gemini-3.1-flash-image` path. The renderers in `DiagramAuditGrid.tsx` and
`DiagramLabellingActivity.tsx` already draw leader lines (bubble -> tip dot) when
`tipX/tipY` exist. The bad screenshots are *old* rows generated before this split.

This plan closes the two remaining gaps with three changes, per the decisions:
**(A) vision text-detection gate + regenerate, (B) Gemini supplies tips only and
code lays out bubbles deterministically so the tip always touches the structure
and bubbles never crowd, (C) save best-effort drafts instead of discarding.**

## Decisions (confirmed with user)

- Text: vision gate + auto-regenerate (keep `gemini-2.5-flash-image` default).
- Pin layout: Gemini returns the **tip only**; code computes bubble positions.
  Priority: leader-line tip must land on the structure; bubbles never cramp.
- On validation failure: save best-effort as `quality_status: 'draft'`.

## Changes

### 1. Text-detection gate — `src/lib/gemini/diagramImage.ts`
- Add `detectImageText(png: Buffer): Promise<boolean>` — one `gemini-2.5-flash`
  vision call with a strict JSON schema `{ hasText: boolean }`, prompt: "Does this
  image contain ANY readable letters, words, or numbers? Ignore purely pictorial
  shapes." Reuse the existing `getGeminiClient()` + `responseSchema` pattern from
  `diagramPoints.ts`.
- In `generateDiagramImage`, loop up to 3 attempts: generate -> `detectImageText`;
  return the first clean image. If all 3 have text, return the last attempt but
  surface a `hadText: true` flag in the return type so the pipeline can mark the
  row `draft`. Keep the existing 3.1/2.5 model branching and size checks.

### 2. Tips-only extraction + deterministic bubble layout
- `src/lib/gemini/diagramPoints.ts`: simplify the schema to `tip` + `label` only
  (drop `anchor`). Tighten the prompt to one job: "Return the precise pixel centre
  of each structure as `tip: [y,x]` (0-1000). The point must sit INSIDE the drawn
  structure, not in empty space." Return parts with `tipX/tipY` set and `x/y`
  left for the layout step. Keep the 503 retry.
- New `src/lib/gemini/diagramLayout.ts`: `layoutAnchors(tips, width, height)`.
  - For each tip, derive a candidate bubble position by pushing radially outward
    from the image centroid through the tip toward the nearest margin (keeps the
    leader line short and pointing back onto the structure).
  - Resolve crowding deterministically: sort tips by angle around the centroid and
    assign evenly spaced slots along the periphery, so two bubbles can never sit
    within the min-distance. Clamp every bubble inside the viewBox margin.
  - Return `DiagramPart[]` with `x/y` = bubble centre, `tipX/tipY` = structure
    point. No Gemini call — placement is pure, testable geometry.

### 3. Pipeline wiring + best-effort save — `src/lib/gemini/bespokePipeline.ts`
- After `extractDiagramPoints`, call `layoutAnchors` to set bubble coords.
- `validateParts`: keep the tip-margin and tip-distance checks (real signal of a
  bad extraction). The anchor-distance check is now guaranteed by layout, so it
  becomes a cheap assertion, not a failure path.
- Replace the `return null` after retries with a **best-effort save**: persist the
  best attempt's row with `quality_status: 'draft'` (also force `draft` when the
  image `hadText`). The diagram then shows in `/diagram-audit` for manual
  **Edit pins** correction instead of vanishing. Only return `null` on hard
  errors (image gen throw, storage/DB error).

### 4. Tests — `src/__tests__/gemini-diagram-pipeline.test.ts` (+ new layout test)
- Update existing tip/anchor expectations to the tips-only shape.
- New unit tests for `layoutAnchors`: N tips -> N bubbles, no two bubbles within
  min distance, every bubble inside margin, each `tipX/tipY` preserved exactly.
- Mock-based test for `detectImageText` (true/false) and the regenerate loop
  (dirty-then-clean returns the clean buffer).
- Keep `buildDiagramImagePrompt` assertions.

## Out of scope
- No schema/migration changes (`parts` JSON already holds `tipX/tipY`).
- No renderer changes — leader lines already draw correctly.
- No change to the static Wikimedia library path.

## Verification
1. `npx jest --testPathPatterns=gemini-diagram-pipeline` — layout + gate unit tests.
2. `npm test` — full suite (other diagram tests must still pass).
3. `npm run lint` and `npm run build` — type/lint clean.
4. End-to-end: `npm run dev`, open `/diagram-audit`, hit **Regenerate** on a draft
   (e.g. alveolus). Confirm on the new row: (a) no text in the image, (b) every
   leader-line tip dot sits on its structure, (c) numbered bubbles are spread with
   no overlap. If a pin is off, **Edit pins** still works as the final safety net.
