# Plan: Replace Brain Lobes with a Recognisable Educational Diagram

## Context

The diagram labelling feature has 28 templates, all stored as hand-coded inline SVG strings in TypeScript. The current brain-lobes diagram is visually primitive (basic ellipses and a few path strokes), and is not recognisable to a student without reading the title or labels. The execution brief says: "Do not continue scaling the current hand-coded primitive SVG style." This plan introduces file-backed diagram assets and replaces the brain-lobes diagram with a recognisable educational illustration, while preserving all existing infrastructure.

---

## 1. How better stored diagrams will be supported

The system will gain a new `assetKind` field that tells the renderer whether to use an inline SVG string or a file stored in the `public/` folder. All 27 existing diagrams stay exactly as they are (inline SVG, no breaking change). Only brain-lobes is changed in this PR.

## 2. SVG file or image file

SVG file. The brief prefers SVG for crisp line art and small file size. PNG/WebP is only justified if the source is substantially better than what SVG can achieve. A carefully drawn lateral brain diagram is well within SVG's capability.

## 3. Where the new asset will be stored

`public/diagrams/brain-lobes.svg`

Next.js serves everything in `public/` as static files, so the browser fetches it as `/diagrams/brain-lobes.svg`. This folder does not exist yet and will be created as part of this PR.

## 4. Which ONE diagram will be replaced

**Brain lobes** (`id: 'brain-lobes'`). It is the top priority in Batch 1 and is the most important for the psychology/neurology use case.

The new SVG will show a realistic lateral (side-view) silhouette of the brain with:
- A smooth, recognisable outer brain contour using bezier curves
- Clearly delineated lobe regions with distinct light fill colours (muted, educational palette matching the app's colour tokens)
- Visible sulcus/gyrus texture suggested by a few internal curves
- Cerebellum at the rear-lower with a distinctly different texture
- Brainstem stub at the bottom

Six parts remain unchanged: Frontal lobe, Parietal lobe, Temporal lobe, Occipital lobe, Cerebellum, Brainstem.

## 5. How label pins will still work

Pin coordinates are **already stored as percentages** once the component converts them. Specifically:

```
xPct = (part.x / viewBoxWidth) * 100
yPct = (part.y / viewBoxHeight) * 100
```

These percentages then set `left` and `top` on absolutely-positioned button elements that float over the diagram container. This works identically whether the underlying content is an inline SVG or an `<img>` tag - both fill the same container. No coordinate system change is needed; only the pin x/y values may need minor tuning after visual review of the new SVG.

## 6. Files to change

| File | Change |
|------|--------|
| `src/lib/diagrams/templates.ts` | Add `assetKind` + optional `imageSrc?` to interface; make `svg` optional; default all existing templates to `assetKind: 'inline-svg'`; update brain-lobes entry to `public-image` |
| `src/lib/types.ts` | Add `assetKind?` and `imageSrc?` to `DiagramLabellingQuestion` (lines 102-109); make `svg` optional |
| `src/components/study/activities/DiagramLabellingActivity.tsx` | Make `svg` prop optional; add `imageSrc?` and `assetKind?` props; render `<img>` for `public-image` kind |
| `src/components/diagrams/DiagramAuditGrid.tsx` | Same rendering update so the audit page shows the image correctly |
| `src/__tests__/diagram-template-registry.test.ts` | Update assertions: every template must have `assetKind`; inline-svg templates must have `svg`; public-image templates must have `imageSrc` |
| `public/diagrams/brain-lobes.svg` | New file — high-quality lateral brain diagram |

No other files need changing. The generate route and SessionRunner pass through the `DiagramLabellingQuestion` shape, which only needs `svg` made optional and `imageSrc?` added.

## 7. How to visually check the result

1. Run `npm run dev` (starts the local development server on port 3000).
2. Open `http://localhost:3000/diagram-audit` in a browser.
3. Find the brain-lobes card. It should now show a recognisable lateral brain with coloured lobe regions.
4. Confirm the six numbered pins each land on the correct lobe/structure.
5. Check on a narrow mobile-width window (375px) to confirm the image and pins still scale correctly.
6. Optionally: open any study set that contains a diagram labelling activity and play through it to confirm the activity works end-to-end.

## 8. Tests and checks to run

```
npm test                  # All 16 Jest suites must stay green
npm run lint              # ESLint must be clean
npm run build             # TypeScript must compile without errors
```

After the above pass, inspect `/diagram-audit` in the browser (step 7 above). Only mark `qualityStatus: 'reviewed'` on the brain-lobes template once the visual check is done.

## 9. Is this safe to push live

Yes. The change is narrowly scoped:
- Only the brain-lobes template switches from inline SVG to a file-backed image.
- All 27 other templates are untouched.
- No database migrations.
- No new runtime dependencies.
- The `/diagram-audit` page will surface any pin placement problems before the PR is merged.
- Worst case: if the new brain-lobes SVG looks wrong, the template's `qualityStatus` stays `'needs-redraw'` and no regression affects other activity types.

The PR notes should include a screenshot from `/diagram-audit` showing the new diagram with pins.
