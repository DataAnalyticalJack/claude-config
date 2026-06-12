# Plan: Delta backfill + full refresh for study sets

## Context

Today every "Regenerate study set" click (folder page, exam prep, and the stale-folder path of "Update all folders") re-runs **all three** generation groups and **inserts a new row**:

- **Core** (Haiku): flashcards, mcq, truefalse, fillinblank, matching
- **Applied** (Sonnet, the expensive one): shortanswer, casestudy
- **Diagrams** (Haiku): diagramlabelling

So a user who only needs the two missing tiles (e.g. case studies + diagrams) pays the full token cost, and duplicate `study_sets` rows accumulate. This plan makes generation gap-aware so the user can cheaply fill only what's missing, optionally refresh regenerable content, and stop creating duplicate rows.

**Decisions locked with the user:**
- Scope = **engine first**. Exam-prep save/resume/delete/list is a **separate follow-up plan** (noted at the end).
- **Full Refresh = core + applied only**; existing diagram labelling is preserved (re-running template matching wastes tokens for the same result).
- Regenerate **updates the existing row in place** when a study set id is supplied; brand-new generation still inserts.

## Approach

Add gap-aware modes to `/api/generate`, share one gap-detection helper between server and client, and surface two clear actions per folder ("Backfill missing" + "Refresh content"). Upgrade "Update all folders" to delta. Exam-prep generation is unchanged in this plan (its resume/save/delete is the follow-up).

### 1. New shared helper — `src/lib/studyset.ts` (new file)

Single source of truth for "what's missing", reused by the API and `BackfillButton` (removes the inline duplicate at [BackfillButton.tsx:58](src/components/project/BackfillButton.tsx#L58)).

```ts
import type { StudySetContent } from '@/lib/types'

const CORE_KEYS = ['flashcards','mcq','truefalse','fillinblank','matching'] as const

export function studySetGaps(content: StudySetContent | null | undefined) {
  const has = (k: keyof StudySetContent) =>
    Array.isArray(content?.[k]) && (content![k] as unknown[]).length > 0
  const core = !content || !CORE_KEYS.every(has)
  const applied = !has('shortanswer') || !has('casestudy')
  const diagrams = content?.diagramlabelling === undefined   // matches today's semantics; [] counts as present
  return { core, applied, diagrams, any: core || applied || diagrams }
}
```

Note: `diagrams` uses `=== undefined` deliberately — an empty `diagramlabelling: []` means "tried, no template matched" and must NOT be treated as a gap (the existing `BackfillButton.test.tsx` `currentSet` relies on this).

### 2. Engine changes — `src/app/api/generate/route.ts`

Extend the request body (backward compatible; existing callers omit the new fields):

```ts
{ subFolderIds, projectId, mode?: 'full' | 'delta' | 'refresh', studySetId?: string }
```

Behaviour in the `POST` handler ([route.ts:215](src/app/api/generate/route.ts#L215)):

- **No `studySetId`** (default / `mode: 'full'`): unchanged — generate all three groups, **INSERT** new row.
- **`studySetId` present**: fetch that row's `content_json` first. Decide which groups to run:
  - `mode: 'delta'` → run only the groups flagged by `studySetGaps(existing)` (`core`, `applied`, `diagrams`). If `!gaps.any`, short-circuit: return the existing row untouched, **zero API calls**.
  - `mode: 'refresh'` → always run **core + applied**, never diagrams.
  - Build the three promises **conditionally** (replace the unconditional `Promise.all` at [route.ts:250-260](src/app/api/generate/route.ts#L250)); a skipped group resolves to `null`.
  - Merge over existing content, then **UPDATE** the row:
    ```ts
    const content = {
      ...existing,
      ...(core ?? {}),
      ...(applied ?? {}),
      ...(diagrams !== null ? { diagramlabelling: diagrams } : {}),
    }
    await supabase.from('study_sets').update({ content_json: content }).eq('id', studySetId)
    ```
  - Preserve the existing `warnings` behaviour for any group that was attempted and failed.

Materials fetch / `sharedContent` build only needs to happen when at least one group will run (skip work on the delta short-circuit).

### 3. Folder page UI — `src/app/projects/[id]/[folderId]/page.tsx`

Replace the single "Regenerate study set" link ([page.tsx:176-181](src/app/projects/[id]/[folderId]/page.tsx#L176)) with two gap-aware actions, reusing the existing `GeneratingBar` and warm token styling:

- Compute `const gaps = studySetGaps(studySet?.content_json)`.
- **Backfill** (only when `gaps.any`): a small amber note naming missing tiles (e.g. "Missing: Case Studies, Diagram Labelling") + a **"Backfill missing"** button → `generate('delta')`.
- **"Refresh content"** quiet link (always, when a set exists) → `generate('refresh')`.
- Generalize the existing `generate()` to take a mode and pass `mode` + `studySetId: studySet.id`; on success set state from the returned/updated row so tiles update **without a full reload**. Brand-new generation (no set yet) keeps calling with no mode/id.

### 4. "Update all folders" — `src/components/project/BackfillButton.tsx`

- Replace inline `isStale` ([line 58](src/components/project/BackfillButton.tsx#L58)) with `studySetGaps(data?.content_json).any`.
- In the regenerate call ([line 88-92](src/components/project/BackfillButton.tsx#L88)) pass `mode: 'delta', studySetId: data.id` so stale folders only fill gaps instead of full-regenerating. (Keep the existing 2-way concurrency and per-folder progress UI.)

### 5. Exam Prep — unchanged this plan

`exam-prep/page.tsx` generation stays as-is (still creates a fresh set). Resume/save/delete/refresh for exam prep — including delta/refresh on a resumed set — is the **follow-up plan** (needs a way to detect an existing multi-folder set by exact `source_subfolder_ids`, a `DELETE /api/study-sets/[id]` endpoint, and a list/resume UI).

## Files to touch

- `src/lib/studyset.ts` — **new**, `studySetGaps` helper.
- `src/app/api/generate/route.ts` — modes, conditional generation, update-in-place.
- `src/app/projects/[id]/[folderId]/page.tsx` — two-action UI, mode-aware `generate()`.
- `src/components/project/BackfillButton.tsx` — use helper + pass delta mode/id.
- `src/__tests__/studyset.test.ts` — **new**, unit tests for `studySetGaps`.
- `src/__tests__/BackfillButton.test.tsx` — update mock assertions for the new request body (`mode`/`studySetId`); existing `currentSet`/`staleSet`/`nodiagram` cases must still pass.

## Verification

- `npx jest --testPathPatterns=studyset` and `--testPathPatterns=BackfillButton` (Jest 30 flag).
- `npm run build` for type check across the new request shape.
- Manual (dev): open a folder whose set lacks case studies/diagrams → confirm "Missing: ..." note + Backfill fills only those tiles and the row id is unchanged (no duplicate). Then "Refresh content" → core+applied change, diagrams preserved. Then "Update all folders" → only gap folders run.

## Out of scope (follow-up plan)

Exam-prep persistence: detect/resume an existing multi-folder set, named save, `DELETE /api/study-sets/[id]`, and a "previous exam prep sets" list with delta/refresh actions.
