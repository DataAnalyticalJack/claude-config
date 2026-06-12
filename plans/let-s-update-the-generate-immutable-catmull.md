# Context

Three small UX improvements to the Exam Prep feature:
1. The "Regenerate" button label is confusing when a matching set already exists — it should signal navigation, not a destructive re-generation.
2. The Previous Sets cards show a long dot-separated list of folder names which gets cluttered fast.
3. The "Resume" button label doesn't reflect actual session state — users who've never started should see "Begin", users mid-session should see "Resume", and completers should see "Try again".

---

## Change 1 — "→" button when a matching set exists

**File:** `src/app/projects/[id]/exam-prep/page.tsx`

When `matchingSet` is found (selected folders already have a study set), the main button currently labels itself "Regenerate Exam Prep (N folders)" and re-calls the API with `mode: 'refresh'`. Change it to navigate directly to the existing set — no API call needed. The card's "Refresh" button covers the regenerate case.

```tsx
// New click handler (replaces generate() call in the button)
function handleMainButton() {
  if (matchingSet) {
    localStorage.setItem('exam_started_' + matchingSet.id, '1')
    router.push(studyUrl(matchingSet))
  } else {
    generate()
  }
}

// Label
const generateLabel = generating
  ? 'Working...'
  : matchingSet
    ? '→'
    : `Generate Exam Prep (${selected.size} folder${selected.size !== 1 ? 's' : ''})`
```

Add `aria-label="Continue to exam prep"` on the button when `matchingSet` is truthy.

---

## Change 2 — Remove folder names from Previous Sets cards

**File:** `src/components/exam-prep/ExamPrepSetCard.tsx`

Remove line 46:
```tsx
<p className="text-sm font-medium text-text-primary">{set.folder_names.join(' · ')}</p>
```

The date and session count remain. The card will be less cluttered and the "New material" badge stays as the primary differentiator.

---

## Change 3 — Smart button label: Begin / Resume / Try again

**Files:** `src/components/exam-prep/ExamPrepSetCard.tsx`, `src/components/study/SessionRunner.tsx`

Use `localStorage` to track in-progress sessions without any database or API changes. Sessions are only saved on completion, so localStorage is the only way to detect "started but not finished".

### ExamPrepSetCard — read state + write on navigate

```tsx
const [inProgress, setInProgress] = useState(false)

useEffect(() => {
  setInProgress(!!localStorage.getItem('exam_started_' + set.id))
}, [set.id])

const resumeLabel = inProgress
  ? 'Resume'
  : set.session_count > 0
    ? 'Try again'
    : 'Begin'

function handleResume() {
  localStorage.setItem('exam_started_' + set.id, '1')
  onResume()
}
```

Replace `onClick={onResume}` with `onClick={handleResume}` and use `{resumeLabel}` as the button text.

### SessionRunner — clear on save

In `saveSession()`, after the fetch, add:
```tsx
if (typeof window !== 'undefined' && studySetId) {
  localStorage.removeItem('exam_started_' + studySetId)
}
```

This clears the in-progress flag once a session is completed and saved. Safe for non-exam sessions because the key would never have been set.

---

## Files to touch

| File | Change |
|------|--------|
| `src/app/projects/[id]/exam-prep/page.tsx` | New `handleMainButton`, updated label, localStorage write for "→" path |
| `src/components/exam-prep/ExamPrepSetCard.tsx` | Remove folder names; add `inProgress` state; smart label; `handleResume` |
| `src/components/study/SessionRunner.tsx` | Clear `exam_started_*` key in `saveSession` |

---

## Verification

1. `npm run build` — type check passes
2. Manual: select folders with no existing set → button says "Generate Exam Prep (N folders)" → clicking generates
3. Manual: select folders with an existing set → button shows "→" → clicking navigates directly without API call
4. Manual: Previous Sets card — no folder name list visible, only date + session count
5. Manual: Fresh set (no sessions) → card shows "Begin" → click → navigate → come back without completing → card shows "Resume"
6. Manual: Complete a session → come back → card shows "Try again"
