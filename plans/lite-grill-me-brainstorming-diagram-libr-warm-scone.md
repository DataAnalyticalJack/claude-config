# Automated Bespoke Diagram Pipeline (Gemini, catalogue-first)

## Context

The diagram-labelling activity is fully shipped: 25 static templates ([templates.ts](src/lib/diagrams/templates.ts)), a percentage-pin renderer ([DiagramLabellingActivity.tsx](src/components/study/activities/DiagramLabellingActivity.tsx)), Haiku template matching in `/api/generate`, and a dev-only `/diagram-audit` page. The gap: when uploaded content covers a topic with no template (e.g. the ear, the eye), students get nothing.

This change adds an automated pipeline: when no template matches a diagram-worthy topic, the app generates a bespoke clean diagram with Gemini (nano banana), extracts pin coordinates via Gemini vision pointing, sanity-checks, stores it in a shared Supabase catalogue, and serves it - first upload pays the cost, everyone after reuses it. A dev curation tool fixes pins or culls bad images.

## Decisions (grilled & resolved)

- Catalogue-first: check static 25 + new `diagram_templates` DB table; generate only on miss, max 2 per study set
- `gemini-2.5-flash-image` generates ONE clean unlabelled PNG (pastel, university scientific style, 6-8 structures); `gemini-2.5-flash` pointing extracts coords (normalized [y,x] 0-1000 -> viewBox px)
- Live immediately after sanity checks (6-8 parts, in-bounds + margin, min pin spacing); retry pointing once, then discard with warning
- Runs parallel inside `/api/generate`, 90s cap via Promise race + `after()` so late results still land in catalogue; failures use existing `warnings` pattern
- No web grounding v1. Renderer, types.ts, utils.ts unchanged - DB templates flatten to existing `DiagramLabellingQuestion` shape at generate time
- Dev tool: extend `/diagram-audit` (stays 404-in-prod) - click-to-place pin editing + save/delete/regenerate for DB templates; click-logs-coords + clipboard JSON for static ones
- Env: `GEMINI_API_KEY` (AI Studio free tier first), `BESPOKE_DIAGRAMS=false` kill switch, optional `DIAGRAM_ADMIN_KEY`
- SDK: `@google/genai` (GA), lazy-init singleton

## New files

| Path | Purpose |
|---|---|
| `supabase/migrations/20260611120000_diagram_templates.sql` | table + public `diagrams` storage bucket + read-only RLS (writes service-role only) |
| `src/lib/gemini/client.ts` | lazy `GoogleGenAI` singleton, `isGeminiConfigured()` |
| `src/lib/gemini/diagramImage.ts` | `generateDiagramImage()` + `readPngDimensions()` (IHDR parse, no deps) |
| `src/lib/gemini/diagramPoints.ts` | `extractDiagramPoints()` + pure `convertPointToViewBox()` (unit-tested - highest-risk line) |
| `src/lib/gemini/bespokePipeline.ts` | `generateBespokeTemplate()`: dedupe-by-slug -> gen -> point -> `validateParts()` -> upload -> insert; `withTimeout()` |
| `src/lib/diagrams/dbTemplates.ts` | row type, `fetchDbTemplates()`, `dbRowToQuestion()`, `dbRowToSummaryLine()`, `db:` id prefix |
| `src/lib/diagrams/adminGuard.ts` | `isDiagramAdminRequest()` - allow in dev, else require `x-diagram-admin-key` header; fail = 404 |
| `src/app/api/diagram-templates/[id]/route.ts` | PATCH (parts/metadata), DELETE (row + storage object) |
| `src/app/api/diagram-templates/[id]/regenerate/route.ts` | POST - rerun pipeline in replace mode |
| `src/__tests__/gemini-diagram-pipeline.test.ts` | coord conversion, PNG dims, validateParts, retry-once, dedupe (Gemini mocked) |
| `src/__tests__/diagram-templates-api.test.ts` | guard + CRUD tests |

## Modified files

- [src/app/api/generate/route.ts](src/app/api/generate/route.ts) - registry summary becomes per-request `buildRegistrySummary(dbRows)` (replaces module const at line 159; DB lines sit after the `cache_control` block so the Haiku cache prefix survives); `DIAGRAM_TOOL` gains optional `missing_topics` field (maxItems 2, 6-8 concrete structures, explicit "no abstract topics" instruction); `generateDiagrams` resolves `db:`-prefixed IDs; bespoke promise kicked off un-awaited after Phase-1 `Promise.all`, raced at 90s before content assembly, registered with `after()`; gated on existing `runDiagrams` flag (so refresh/delta modes never burn quota)
- [src/app/diagram-audit/page.tsx](src/app/diagram-audit/page.tsx) - async, `force-dynamic`, fetch DB rows via service client
- [src/components/diagrams/DiagramAuditGrid.tsx](src/components/diagrams/DiagramAuditGrid.tsx) - `'use client'`; DB cards: edit-pins mode (arm pin, click image to place), Save -> PATCH, Delete (confirm), Regenerate (spinner -> refresh); static cards: click logs viewBox coords + copies JSON
- `src/__tests__/generate-api.test.ts`, `src/__tests__/DiagramAuditGrid.test.tsx` - mock pipeline module + new cases
- `package.json` - add `@google/genai`

## Migration sketch

`diagram_templates`: `id uuid pk`, `slug text unique` (dedupe anchor), `title`, `description`, `subjects/aliases/keywords text[]`, `difficulty` check, `quality_status` check ('draft'|'reviewed'|'needs-redraw'), `image_path`, `image_url`, `view_box_width/height int` (= PNG pixel dims), `parts jsonb`, `distractors text[]`, `source`, timestamps. RLS: select for anon/authenticated; no write policies (service role bypasses). Storage bucket `diagrams` public-read, no client write policies. Apply: `npx supabase db push`.

## Phasing (3 PRs, each green)

1. **Catalogue read path** - migration, `dbTemplates.ts`, per-request summary, `db:` resolution, audit page read-only DB cards, test updates. Safe with empty table (output identical to today).
2. **Gemini pipeline** - SDK dep, `src/lib/gemini/*`, `missing_topics` tool extension, route wiring (timeout/`after()`/warnings/kill switch), pipeline tests, Vercel env `GEMINI_API_KEY`.
3. **Curation tooling** - admin guard, two API routes, audit grid editing UI, route tests.

## Verification

- Per PR: focused Jest (`npx jest --testPathPatterns=gemini-diagram-pipeline` etc.), then `npm test` + `npm run lint` + `npm run build` before merge
- Manual after PR 2: upload notes on an uncovered topic (e.g. human ear) -> confirm DB card on `/diagram-audit`; **eyeball pin placement first** - a [y,x]/[x,y] swap shows as diagonally-flipped pins (the #1 failure mode); run a study session, confirm bespoke diagram renders in unchanged `DiagramLabellingActivity`
- Manual after PR 3: edit/save/delete/regenerate a DB template from `/diagram-audit`
- Watch `cache_read_input_tokens` logs post-deploy (tool schema change = one-time cache bust, not regression)

## Risks

- Point format variance ([y,x] vs [x,y], 0-1000 vs 0-1): constrain with `responseSchema` + prompt example; conversion isolated in one pure tested function
- Free-tier limits (~low RPM/day for image model): mitigated by <=2/set cap, slug dedupe, retry-once, kill switch; 429 -> warning, never 500
- Concurrent same-topic uploads: unique slug constraint; on conflict fetch winner
- Lazy SDK init mandatory - module-level `new GoogleGenAI()` without key would break builds/tests
