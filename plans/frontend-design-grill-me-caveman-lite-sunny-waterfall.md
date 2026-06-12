# Auth + Per-User Data Separation - Implementation Plan Index

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Supabase email/password auth with open public signup, per-user data separation, a warm pastel landing page, and password reset — without sacrificing app performance or responsiveness.

**Architecture:** 4 sequential phases, each its own branch and PR. The app continues to work throughout (RLS stays permissive until Phase 4; enforcement is app-layer from Phase 2 onward).

**Tech Stack:** Supabase Auth + `@supabase/ssr`, Next.js 16 middleware, Tailwind v4, existing UI primitives.

---

## Phase sequence (strict — do not reorder)

| Phase | Plan file | What it does |
|---|---|---|
| P1 | [auth-p1-schema-and-seed.md](docs/superpowers/plans/2026-06-10-auth-p1-schema-and-seed.md) | Add user_id columns to 6 tables; run seed script to create 2 auth accounts and deep-copy all data |
| P2 | [auth-p2-enforcement.md](docs/superpowers/plans/2026-06-10-auth-p2-enforcement.md) | SSR client + middleware + getUserId helper + auth guard on all 18 API routes |
| P3 | [auth-p3-ui.md](docs/superpowers/plans/2026-06-10-auth-p3-ui.md) | Landing page, register, reset, update-password, auth callback, sign-out, dashboard move to /dashboard |
| P4 | [auth-p4-rls-lockdown.md](docs/superpowers/plans/2026-06-10-auth-p4-rls-lockdown.md) | Replace allow_all RLS policies with owner policies; harden storage writes to authenticated-only |

## Design spec

`docs/superpowers/specs/2026-06-10-auth-and-data-separation-design.md`
