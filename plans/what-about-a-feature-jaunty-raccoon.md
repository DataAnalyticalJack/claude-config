# Project Sharing via Private Link — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let project owners generate a private shareable link; recipients can browse the project's subfolders and run study sessions without signing in, with no data copied.

**Architecture:** A new `project_shares` table maps a random UUID token to a project + owner. Three new API routes handle create/get/revoke (owner-only) and a public read-by-token endpoint. Two new pages (`/shared/[token]` and `/shared/[token]/study`) serve visitors. `SessionRunner` gets one optional `onComplete` prop so the shared study page can redirect back to `/shared/[token]` on finish instead of a private project route.

**Tech Stack:** Next.js 16 App Router, TypeScript, Supabase (service role, permissive RLS), Tailwind v4, existing `SessionRunner`, `lucide-react`.

---

## File Map

| Action | Path |
|--------|------|
| Create | `supabase/migrations/20260612000001_project_shares.sql` |
| Edit | `src/lib/types.ts` — append `ProjectShare` interface |
| Create | `src/app/api/projects/[id]/share/route.ts` — GET, POST, DELETE |
| Create | `src/app/api/shared/[token]/route.ts` — public GET |
| Create | `src/app/shared/[token]/page.tsx` — shared landing page |
| Create | `src/app/shared/[token]/study/page.tsx` — shared study session page |
| Create | `src/components/shared/SharedSessionRunner.tsx` — client wrapper |
| Create | `src/components/project/SharePanel.tsx` — owner share UI |
| Edit | `src/components/study/SessionRunner.tsx` — add optional `onComplete` prop |
| Edit | `src/app/projects/[id]/page.tsx` — add share state + render SharePanel |
| Edit | `src/middleware.ts` — add `/shared` to PUBLIC_PATHS |

---

## Task 1 — Database Migration

**Files:**
- Create: `supabase/migrations/20260612000001_project_shares.sql`

- [ ] **Step 1: Write the migration file**

```sql
create table public.project_shares (
  id            uuid primary key default gen_random_uuid(),
  project_id    uuid not null references public.projects(id) on delete cascade,
  owner_user_id uuid not null references auth.users(id) on delete cascade,
  token         text not null unique,
  is_public     boolean not null default false,
  created_at    timestamptz not null default now()
);

create index idx_project_shares_token      on public.project_shares(token);
create index idx_project_shares_project_id on public.project_shares(project_id);

alter table public.project_shares enable row level security;

create policy "project_shares_allow_all" on public.project_shares
  for all using (true) with check (true);
```

- [ ] **Step 2: Apply migration**

```powershell
npx supabase db push
```

Expected: migration applied successfully, `project_shares` table visible in Supabase dashboard.

- [ ] **Step 3: Commit**

```bash
git add supabase/migrations/20260612000001_project_shares.sql
git commit -m "feat(db): add project_shares table with permissive RLS"
```

---

## Task 2 — TypeScript Interface

**Files:**
- Modify: `src/lib/types.ts` (append at end of file)

- [ ] **Step 1: Write the failing test** (`src/__tests__/types.test.ts` — skip if no type-only tests exist in this repo; the interface is verified by the API tests in Task 3)

*This task has no runtime behaviour — the interface is verified by TypeScript compilation. Proceed directly to implementation.*

- [ ] **Step 2: Append interface to `src/lib/types.ts`**

```typescript
export interface ProjectShare {
  id: string
  project_id: string
  token: string
  is_public: boolean
  created_at: string
  // owner_user_id intentionally omitted — never sent to the client
}
```

---

## Task 3 — Owner API Route (GET / POST / DELETE)

**Files:**
- Create: `src/app/api/projects/[id]/share/route.ts`

This sits alongside the existing `src/app/api/projects/[id]/route.ts`. Next.js App Router resolves them independently.

- [ ] **Step 1: Write failing tests** (`src/__tests__/share-api.test.ts`)

Follow the pattern of `src/__tests__/subfolders-api.test.ts` (mock `@/lib/supabase/server`, mock `@/lib/auth/getUserId`).

```typescript
import { GET, POST, DELETE } from '@/app/api/projects/[id]/share/route'

const mockFrom = jest.fn()
jest.mock('@/lib/supabase/server', () => ({
  createServiceClient: () => ({ from: mockFrom }),
}))
jest.mock('@/lib/auth/getUserId', () => ({ getUserId: jest.fn() }))
import { getUserId } from '@/lib/auth/getUserId'

const makeParams = (id: string) => ({ params: Promise.resolve({ id }) })

describe('GET /api/projects/[id]/share', () => {
  it('returns 401 when not authenticated', async () => {
    (getUserId as jest.Mock).mockResolvedValue(null)
    const res = await GET(new Request('http://x'), makeParams('p1'))
    expect(res.status).toBe(401)
  })

  it('returns null when no share exists', async () => {
    (getUserId as jest.Mock).mockResolvedValue('u1')
    mockFrom.mockReturnValue({
      select: () => ({ eq: () => ({ eq: () => ({ maybeSingle: () => ({ data: null }) }) }) }),
    })
    const res = await GET(new Request('http://x'), makeParams('p1'))
    expect(await res.json()).toBeNull()
  })

  it('returns share data when share exists', async () => {
    (getUserId as jest.Mock).mockResolvedValue('u1')
    const share = { id: 's1', project_id: 'p1', token: 'tok', is_public: false, created_at: '' }
    mockFrom.mockReturnValue({
      select: () => ({ eq: () => ({ eq: () => ({ maybeSingle: () => ({ data: share }) }) }) }),
    })
    const res = await GET(new Request('http://x'), makeParams('p1'))
    expect(await res.json()).toEqual(share)
  })
})

describe('POST /api/projects/[id]/share', () => {
  it('returns 401 when not authenticated', async () => {
    (getUserId as jest.Mock).mockResolvedValue(null)
    const res = await POST(new Request('http://x'), makeParams('p1'))
    expect(res.status).toBe(401)
  })

  it('returns 404 when project not found for this user', async () => {
    (getUserId as jest.Mock).mockResolvedValue('u1')
    mockFrom.mockReturnValue({
      select: () => ({ eq: () => ({ eq: () => ({ single: () => ({ data: null, error: new Error('not found') }) }) }) }),
    })
    const res = await POST(new Request('http://x'), makeParams('p1'))
    expect(res.status).toBe(404)
  })

  it('returns existing share with 200 when one already exists', async () => {
    (getUserId as jest.Mock).mockResolvedValue('u1')
    const existing = { id: 's1', project_id: 'p1', token: 'tok', is_public: false, created_at: '' }
    let callCount = 0
    mockFrom.mockImplementation(() => ({
      select: () => ({ eq: () => ({ eq: () => ({
        single: () => callCount++ === 0
          ? { data: { id: 'p1' }, error: null }
          : { data: existing },
      }) }) }),
    }))
    const res = await POST(new Request('http://x'), makeParams('p1'))
    expect(res.status).toBe(200)
    expect((await res.json()).token).toBe('tok')
  })
})

describe('DELETE /api/projects/[id]/share', () => {
  it('returns 401 when not authenticated', async () => {
    (getUserId as jest.Mock).mockResolvedValue(null)
    const res = await DELETE(new Request('http://x'), makeParams('p1'))
    expect(res.status).toBe(401)
  })

  it('returns 204 on successful revoke', async () => {
    (getUserId as jest.Mock).mockResolvedValue('u1')
    mockFrom.mockReturnValue({
      delete: () => ({ eq: () => ({ eq: () => ({ error: null }) }) }),
    })
    const res = await DELETE(new Request('http://x'), makeParams('p1'))
    expect(res.status).toBe(204)
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```powershell
npx jest --testPathPatterns=share-api
```

Expected: FAIL (module not found)

- [ ] **Step 3: Create `src/app/api/projects/[id]/share/route.ts`**

```typescript
import { NextResponse } from 'next/server'
import { createServiceClient } from '@/lib/supabase/server'
import { getUserId } from '@/lib/auth/getUserId'

export async function GET(
  _req: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const userId = await getUserId()
  if (!userId) return NextResponse.json({ error: 'unauthorized' }, { status: 401 })

  const supabase = createServiceClient()
  const { id: projectId } = await params

  const { data } = await supabase
    .from('project_shares')
    .select('id, project_id, token, is_public, created_at')
    .eq('project_id', projectId)
    .eq('owner_user_id', userId)
    .maybeSingle()

  return NextResponse.json(data ?? null)
}

export async function POST(
  _req: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const userId = await getUserId()
  if (!userId) return NextResponse.json({ error: 'unauthorized' }, { status: 401 })

  const supabase = createServiceClient()
  const { id: projectId } = await params

  const { data: project, error: projectError } = await supabase
    .from('projects')
    .select('id')
    .eq('id', projectId)
    .eq('user_id', userId)
    .single()

  if (projectError || !project) {
    return NextResponse.json({ error: 'not found' }, { status: 404 })
  }

  const { data: existing } = await supabase
    .from('project_shares')
    .select('id, project_id, token, is_public, created_at')
    .eq('project_id', projectId)
    .eq('owner_user_id', userId)
    .single()

  if (existing) return NextResponse.json(existing)

  const token = crypto.randomUUID()

  const { data: share, error: insertError } = await supabase
    .from('project_shares')
    .insert({ project_id: projectId, owner_user_id: userId, token, is_public: false })
    .select('id, project_id, token, is_public, created_at')
    .single()

  if (insertError) return NextResponse.json({ error: insertError.message }, { status: 500 })

  return NextResponse.json(share, { status: 201 })
}

export async function DELETE(
  _req: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const userId = await getUserId()
  if (!userId) return NextResponse.json({ error: 'unauthorized' }, { status: 401 })

  const supabase = createServiceClient()
  const { id: projectId } = await params

  const { error } = await supabase
    .from('project_shares')
    .delete()
    .eq('project_id', projectId)
    .eq('owner_user_id', userId)

  if (error) return NextResponse.json({ error: error.message }, { status: 500 })

  return new NextResponse(null, { status: 204 })
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```powershell
npx jest --testPathPatterns=share-api
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/app/api/projects/[id]/share/route.ts src/__tests__/share-api.test.ts src/lib/types.ts
git commit -m "feat(api): owner share endpoints — GET/POST/DELETE /api/projects/[id]/share"
```

---

## Task 4 — Public Shared-Content API

**Files:**
- Create: `src/app/api/shared/[token]/route.ts`
- Create: `src/__tests__/shared-api.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
import { GET } from '@/app/api/shared/[token]/route'

const mockFrom = jest.fn()
jest.mock('@/lib/supabase/server', () => ({
  createServiceClient: () => ({ from: mockFrom }),
}))

const makeParams = (token: string) => ({ params: Promise.resolve({ token }) })

describe('GET /api/shared/[token]', () => {
  it('returns 404 for invalid token', async () => {
    mockFrom.mockReturnValue({
      select: () => ({ eq: () => ({ single: () => ({ data: null, error: new Error('no row') }) }) }),
    })
    const res = await GET(new Request('http://x'), makeParams('bad-token'))
    expect(res.status).toBe(404)
  })

  it('returns project, subfolders, and study_set metadata for valid token', async () => {
    let call = 0
    mockFrom.mockImplementation(() => {
      call++
      if (call === 1) return { select: () => ({ eq: () => ({ single: () => ({ data: { project_id: 'p1', owner_user_id: 'u1' }, error: null }) }) }) }
      if (call === 2) return { select: () => ({ eq: () => ({ eq: () => ({ single: () => ({ data: { id: 'p1', name: 'Bio', colour: '#6B8F71' }, error: null }) }) }) }) }
      if (call === 3) return { select: () => ({ eq: () => ({ eq: () => ({ order: () => ({ data: [{ id: 'sf1', name: 'Week 1', sort_order: 0 }], error: null }) }) }) }) }
      return { select: () => ({ eq: () => ({ is: () => ({ order: () => ({ data: [{ id: 'ss1', source_subfolder_ids: ['sf1'], generated_at: '2026-06-11T00:00:00Z' }] }) }) }) }) }
    })
    const res = await GET(new Request('http://x'), makeParams('tok'))
    const body = await res.json()
    expect(body.project.name).toBe('Bio')
    expect(body.subfolders[0].study_set.id).toBe('ss1')
    expect(body).not.toHaveProperty('owner_user_id')
    expect(body.subfolders[0]).not.toHaveProperty('content_json')
  })

  it('returns study_set: null for subfolders with no study set', async () => {
    let call = 0
    mockFrom.mockImplementation(() => {
      call++
      if (call === 1) return { select: () => ({ eq: () => ({ single: () => ({ data: { project_id: 'p1', owner_user_id: 'u1' }, error: null }) }) }) }
      if (call === 2) return { select: () => ({ eq: () => ({ eq: () => ({ single: () => ({ data: { id: 'p1', name: 'Bio', colour: '#6B8F71' }, error: null }) }) }) }) }
      if (call === 3) return { select: () => ({ eq: () => ({ eq: () => ({ order: () => ({ data: [{ id: 'sf1', name: 'Week 1', sort_order: 0 }], error: null }) }) }) }) }
      return { select: () => ({ eq: () => ({ is: () => ({ order: () => ({ data: [] }) }) }) }) }
    })
    const res = await GET(new Request('http://x'), makeParams('tok'))
    const body = await res.json()
    expect(body.subfolders[0].study_set).toBeNull()
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```powershell
npx jest --testPathPatterns=shared-api
```

Expected: FAIL (module not found)

- [ ] **Step 3: Create `src/app/api/shared/[token]/route.ts`**

```typescript
import { NextResponse } from 'next/server'
import { createServiceClient } from '@/lib/supabase/server'

export async function GET(
  _req: Request,
  { params }: { params: Promise<{ token: string }> }
) {
  const supabase = createServiceClient()
  const { token } = await params

  const { data: share, error: shareError } = await supabase
    .from('project_shares')
    .select('project_id, owner_user_id')
    .eq('token', token)
    .single()

  if (shareError || !share) {
    return NextResponse.json({ error: 'share not found' }, { status: 404 })
  }

  const { project_id: projectId, owner_user_id: ownerId } = share

  const { data: project, error: projectError } = await supabase
    .from('projects')
    .select('id, name, colour')
    .eq('id', projectId)
    .eq('user_id', ownerId)
    .single()

  if (projectError || !project) {
    return NextResponse.json({ error: 'project not found' }, { status: 404 })
  }

  const { data: subfolders, error: subfoldersError } = await supabase
    .from('subfolders')
    .select('id, name, sort_order')
    .eq('project_id', projectId)
    .eq('user_id', ownerId)
    .order('sort_order', { ascending: true })

  if (subfoldersError) {
    return NextResponse.json({ error: subfoldersError.message }, { status: 500 })
  }

  const subfolderIds = (subfolders ?? []).map(s => s.id)
  const studySetMap: Record<string, { id: string; generated_at: string }> = {}

  if (subfolderIds.length > 0) {
    const { data: studySets } = await supabase
      .from('study_sets')
      .select('id, source_subfolder_ids, generated_at')
      .eq('user_id', ownerId)
      .is('project_id', null)
      .order('generated_at', { ascending: false })

    for (const ss of studySets ?? []) {
      if (ss.source_subfolder_ids.length === 1) {
        const fid = ss.source_subfolder_ids[0]
        if (subfolderIds.includes(fid) && !studySetMap[fid]) {
          studySetMap[fid] = { id: ss.id, generated_at: ss.generated_at }
        }
      }
    }
  }

  const enrichedSubfolders = (subfolders ?? []).map(sf => ({
    ...sf,
    study_set: studySetMap[sf.id] ?? null,
  }))

  return NextResponse.json({
    project: { id: project.id, name: project.name, colour: project.colour },
    subfolders: enrichedSubfolders,
  })
}
```

- [ ] **Step 4: Run tests**

```powershell
npx jest --testPathPatterns=shared-api
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/app/api/shared/[token]/route.ts src/__tests__/shared-api.test.ts
git commit -m "feat(api): public GET /api/shared/[token] for shared project data"
```

---

## Task 5 — SessionRunner `onComplete` prop

**Files:**
- Modify: `src/components/study/SessionRunner.tsx`

The current `finishSession` in `SessionRunner` calls `router.push('/projects/${projectId}')` when `folderId`/`studySetId` are null. Adding `onComplete?: () => void` lets the shared study page override this navigation cleanly without modifying any existing call site.

- [ ] **Step 1: Locate the interface and `finishSession` in `src/components/study/SessionRunner.tsx`**

Current interface (line ~18):
```typescript
interface SessionRunnerProps {
  projectId: string
  folderId?: string | null
  studySetId?: string | null
  content: StudySetContent
  activityFilter: string
  questionLimit?: number
}
```

Current `finishSession` (line ~217):
```typescript
function finishSession(finalResults: SessionResult[]) {
  if (folderId && studySetId) {
    saveSession(finalResults)
  } else {
    router.push(`/projects/${projectId}`)
  }
}
```

- [ ] **Step 2: Write a failing test addition in `src/__tests__/SessionRunner.test.tsx`**

Find the existing test file. Add one test:

```typescript
it('calls onComplete instead of router.push when provided and folderId is null', async () => {
  const onComplete = jest.fn()
  // render SessionRunner with folderId=null, studySetId=null, onComplete provided
  // simulate completing all questions
  // assert onComplete was called and router.push was NOT called
  render(
    <SessionRunner
      projectId="p1"
      folderId={null}
      studySetId={null}
      content={mockContent}
      activityFilter="mixed"
      onComplete={onComplete}
    />
  )
  // answer all questions — use existing test helpers from the file
  // ... (adapt to whatever pattern the file uses)
  expect(onComplete).toHaveBeenCalled()
})
```

Run to confirm fail:
```powershell
npx jest --testPathPatterns=SessionRunner
```

- [ ] **Step 3: Apply the minimal change to `SessionRunner.tsx`**

Add `onComplete?: () => void` to the interface and destructure it. Update `finishSession`:

```typescript
interface SessionRunnerProps {
  projectId: string
  folderId?: string | null
  studySetId?: string | null
  content: StudySetContent
  activityFilter: string
  questionLimit?: number
  onComplete?: () => void
}

// In the component body, add to destructuring:
// onComplete,

function finishSession(finalResults: SessionResult[]) {
  if (folderId && studySetId) {
    saveSession(finalResults)
  } else if (onComplete) {
    onComplete()
  } else {
    router.push(`/projects/${projectId}`)
  }
}
```

- [ ] **Step 4: Run tests**

```powershell
npx jest --testPathPatterns=SessionRunner
```

Expected: PASS (all existing + new test)

- [ ] **Step 5: Commit**

```bash
git add src/components/study/SessionRunner.tsx src/__tests__/SessionRunner.test.tsx
git commit -m "feat(SessionRunner): add optional onComplete prop for custom finish navigation"
```

---

## Task 6 — SharedSessionRunner Client Wrapper

**Files:**
- Create: `src/components/shared/SharedSessionRunner.tsx`

This `'use client'` wrapper passes `onComplete` to `SessionRunner` to redirect back to `/shared/[token]` on finish, and suppresses session saving by leaving `folderId`/`studySetId` null.

- [ ] **Step 1: Create `src/components/shared/SharedSessionRunner.tsx`**

```typescript
'use client'
import { useRouter } from 'next/navigation'
import SessionRunner from '@/components/study/SessionRunner'
import type { StudySetContent } from '@/lib/types'

interface SharedSessionRunnerProps {
  token: string
  content: StudySetContent
  activityFilter: string
}

export default function SharedSessionRunner({
  token,
  content,
  activityFilter,
}: SharedSessionRunnerProps) {
  const router = useRouter()

  return (
    <SessionRunner
      projectId=""
      folderId={null}
      studySetId={null}
      content={content}
      activityFilter={activityFilter}
      onComplete={() => router.push(`/shared/${token}`)}
    />
  )
}
```

---

## Task 7 — Shared Pages

**Files:**
- Create: `src/app/shared/[token]/page.tsx`
- Create: `src/app/shared/[token]/study/page.tsx`

- [ ] **Step 1: Create `src/app/shared/[token]/page.tsx`**

```typescript
import { notFound } from 'next/navigation'
import Link from 'next/link'

interface SharedSubfolder {
  id: string
  name: string
  study_set: { id: string; generated_at: string } | null
}

interface Props {
  params: Promise<{ token: string }>
}

export default async function SharedProjectPage({ params }: Props) {
  const { token } = await params

  const res = await fetch(
    `${process.env.NEXT_PUBLIC_APP_URL}/api/shared/${token}`,
    { cache: 'no-store' }
  )

  if (!res.ok) notFound()

  const { project, subfolders } = await res.json()

  return (
    <div className="min-h-screen bg-parchment px-4 py-8 max-w-2xl mx-auto">
      <p className="text-xs text-text-muted uppercase tracking-wider font-semibold mb-1">
        Shared project
      </p>
      <h1 className="text-xl font-bold text-text-primary mb-1">{project.name}</h1>
      <p className="text-sm text-text-muted mb-6">
        Choose a topic to start studying.
      </p>

      <div className="flex flex-col gap-3">
        {(subfolders as SharedSubfolder[]).length === 0 && (
          <p className="text-sm text-text-muted">No topics available yet.</p>
        )}
        {(subfolders as SharedSubfolder[]).map(sf => (
          <div
            key={sf.id}
            className="bg-white border border-sand rounded-xl px-4 py-3 flex items-center justify-between"
          >
            <div>
              <p className="text-sm font-medium text-text-primary">{sf.name}</p>
              {sf.study_set ? (
                <p className="text-xs text-text-muted mt-0.5">
                  Ready — {new Date(sf.study_set.generated_at).toLocaleDateString()}
                </p>
              ) : (
                <p className="text-xs text-text-muted mt-0.5">No study set yet</p>
              )}
            </div>
            {sf.study_set ? (
              <Link
                href={`/shared/${token}/study?subFolderId=${sf.id}&studySetId=${sf.study_set.id}`}
                className="px-4 py-2 bg-sage text-white text-sm font-medium rounded-lg hover:opacity-90 transition-opacity flex-shrink-0"
              >
                Study
              </Link>
            ) : (
              <span className="text-xs text-text-muted px-4 py-2">Not available</span>
            )}
          </div>
        ))}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create `src/app/shared/[token]/study/page.tsx`**

```typescript
import { notFound } from 'next/navigation'
import { createServiceClient } from '@/lib/supabase/server'
import type { StudySet } from '@/lib/types'
import SharedSessionRunner from '@/components/shared/SharedSessionRunner'

interface Props {
  params: Promise<{ token: string }>
  searchParams: Promise<{ subFolderId: string; studySetId: string; activity?: string }>
}

export default async function SharedStudyPage({ params, searchParams }: Props) {
  const supabase = createServiceClient()
  const { token } = await params
  const { subFolderId, studySetId, activity } = await searchParams

  const { data: share } = await supabase
    .from('project_shares')
    .select('project_id, owner_user_id')
    .eq('token', token)
    .single()

  if (!share) notFound()

  const { data: studySet } = await supabase
    .from('study_sets')
    .select('*')
    .eq('id', studySetId)
    .eq('user_id', share.owner_user_id)
    .single()

  if (!studySet) notFound()

  const { data: subfolder } = await supabase
    .from('subfolders')
    .select('name')
    .eq('id', subFolderId)
    .eq('project_id', share.project_id)
    .eq('user_id', share.owner_user_id)
    .single()

  return (
    <div className="min-h-screen bg-parchment px-4 py-8 max-w-2xl mx-auto">
      <h1 className="text-lg font-bold text-text-primary mb-6">
        {subfolder?.name ?? 'Study Session'}
      </h1>
      <SharedSessionRunner
        token={token}
        content={(studySet as StudySet).content_json}
        activityFilter={activity ?? 'mixed'}
      />
    </div>
  )
}
```

- [ ] **Step 3: Add `NEXT_PUBLIC_APP_URL` to `.env.local`**

```
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Also add it to Vercel environment variables (production URL, e.g. `https://your-app.vercel.app`).

- [ ] **Step 4: Update `src/middleware.ts` — add `/shared` to PUBLIC_PATHS**

Current line ~5:
```typescript
const PUBLIC_PATHS = ['/', '/register', '/auth/callback', '/auth/reset', '/auth/update-password']
```

Change to:
```typescript
const PUBLIC_PATHS = ['/', '/register', '/auth/callback', '/auth/reset', '/auth/update-password', '/shared']
```

The existing `startsWith(p + '/')` check at line ~21 means `/shared/anything` is also public.

- [ ] **Step 5: Commit**

```bash
git add src/app/shared/ src/components/shared/SharedSessionRunner.tsx src/middleware.ts
git commit -m "feat(shared): public shared project page and study session"
```

---

## Task 8 — SharePanel Component + Project Page Wiring

**Files:**
- Create: `src/components/project/SharePanel.tsx`
- Modify: `src/app/projects/[id]/page.tsx`

- [ ] **Step 1: Create `src/components/project/SharePanel.tsx`**

```typescript
'use client'
import { useState } from 'react'
import { Link2, Copy, Check, Trash2 } from 'lucide-react'
import type { ProjectShare } from '@/lib/types'

interface SharePanelProps {
  projectId: string
  initialShare: ProjectShare | null
}

export default function SharePanel({ projectId, initialShare }: SharePanelProps) {
  const [share, setShare] = useState<ProjectShare | null>(initialShare)
  const [loading, setLoading] = useState(false)
  const [copied, setCopied] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const shareUrl = share
    ? `${typeof window !== 'undefined' ? window.location.origin : ''}/shared/${share.token}`
    : null

  async function handleShare() {
    setLoading(true)
    setError(null)
    const res = await fetch(`/api/projects/${projectId}/share`, { method: 'POST' })
    if (!res.ok) { setError('Could not create share link.'); setLoading(false); return }
    setShare(await res.json())
    setLoading(false)
  }

  async function handleRevoke() {
    setLoading(true)
    setError(null)
    const res = await fetch(`/api/projects/${projectId}/share`, { method: 'DELETE' })
    if (!res.ok) { setError('Could not revoke link.'); setLoading(false); return }
    setShare(null)
    setLoading(false)
  }

  async function handleCopy() {
    if (!shareUrl) return
    await navigator.clipboard.writeText(shareUrl)
    setCopied(true)
    setTimeout(() => setCopied(false), 2000)
  }

  return (
    <div className="mt-6 bg-white border border-sand rounded-xl px-4 py-4">
      <div className="flex items-center gap-2 mb-3">
        <Link2 className="h-4 w-4 text-text-muted" aria-hidden />
        <p className="text-sm font-semibold text-text-primary">Share</p>
      </div>

      {error && <p className="text-xs text-red-400 mb-3">{error}</p>}

      {!share ? (
        <div>
          <p className="text-xs text-text-muted mb-3">
            Generate a private link so others can study your materials without needing an account.
          </p>
          <button
            onClick={handleShare}
            disabled={loading}
            className="px-4 py-2 bg-sage text-white text-sm font-medium rounded-lg hover:opacity-90 transition-opacity disabled:opacity-50"
          >
            {loading ? 'Creating...' : 'Create share link'}
          </button>
        </div>
      ) : (
        <div className="flex flex-col gap-3">
          <div className="flex items-center gap-2 bg-parchment border border-sand rounded-lg px-3 py-2">
            <span className="text-xs text-text-primary truncate flex-1 font-mono select-all">
              {shareUrl}
            </span>
            <button
              onClick={handleCopy}
              className="flex-shrink-0 text-text-muted hover:text-sage transition-colors"
              title="Copy link"
              aria-label="Copy share link"
            >
              {copied ? <Check className="h-4 w-4 text-sage" /> : <Copy className="h-4 w-4" />}
            </button>
          </div>
          <div className="flex items-center justify-between">
            <p className="text-xs text-text-muted">Anyone with this link can study (no sign-in needed)</p>
            <button
              onClick={handleRevoke}
              disabled={loading}
              className="flex items-center gap-1 text-xs text-text-muted hover:text-red-400 transition-colors disabled:opacity-50"
              aria-label="Revoke share link"
            >
              <Trash2 className="h-3.5 w-3.5" aria-hidden />
              Revoke
            </button>
          </div>
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 2: Update `src/app/projects/[id]/page.tsx`**

Add `ProjectShare` to the existing types import, add a `share` state variable, extend the existing `Promise.all` fetch to also call `GET /api/projects/${id}/share`, and render `<SharePanel>` below the existing content.

Add to imports:
```typescript
import SharePanel from '@/components/project/SharePanel'
// ProjectShare added to existing import from '@/lib/types'
```

Add state:
```typescript
const [share, setShare] = useState<ProjectShare | null>(null)
```

Extend the existing `Promise.all` (find it by the pattern `fetch(\`/api/projects/${id}\`)`):
```typescript
Promise.all([
  fetch(`/api/projects/${id}`).then(r => r.json()),
  fetch(`/api/subfolders?projectId=${id}`).then(r => r.json()),
  fetch(`/api/projects/${id}/share`).then(r => r.ok ? r.json() : null),
]).then(([p, s, sh]) => {
  if (p?.id) setProject(p)
  if (Array.isArray(s)) setSubfolders(s)
  setShare(sh)
  setLoading(false)
})
```

Render SharePanel at the bottom of the main content area:
```tsx
<SharePanel projectId={id} initialShare={share} />
```

- [ ] **Step 3: Run all tests, lint, and build**

```powershell
npm test
npm run lint
npm run build
```

Expected: all tests pass, lint clean (except the pre-existing `DraggableDashboard.tsx:102` error), build succeeds.

- [ ] **Step 4: Commit**

```bash
git add src/components/project/SharePanel.tsx src/app/projects/[id]/page.tsx
git commit -m "feat(ui): share panel on project detail page"
```

---

## Verification Checklist (manual)

1. Run migration — `project_shares` table visible in Supabase dashboard.
2. Sign in as User A, open a project — SharePanel renders with "Create share link".
3. Click "Create share link" — URL appears, clipboard copy works, row appears in `project_shares`.
4. Open the share URL in an incognito window — project name + subfolder list visible, no redirect to sign-in.
5. Click "Study" on a subfolder with a study set — session runs correctly.
6. Complete the session — page returns to `/shared/[token]`, not a 404.
7. Back on the project page, click "Revoke" — SharePanel resets; the previously working URL now returns 404/not-found.
8. Sign in as User B, open User A's share URL — session works; User B's own projects are not visible.
9. Confirm `GET /api/shared/[token]` response body has no `owner_user_id` field.
10. Verify `NEXT_PUBLIC_APP_URL` is set in both `.env.local` and Vercel env vars.
