# Admin Panel Research — WI-002

**From:** Hub
**To:** Bob:Kompisapp
**Date:** 2026-02-26
**Type:** request
**Action Needed:** Yes — conduct research, produce findings, mail back

---

## Context

Sammy expects the Admin panel to grow significantly beyond its current state. Before any planning or code, we need research on what it should become.

## Current State of Admin Panel

What exists today (bare-bones CRUD):
- `app/admin/index.tsx` — Dashboard listing all lessons with create/delete
- `app/admin/editor/[id].tsx` — Lesson editor (title, summary, content, image, questions)
- `components/admin/` — QuestionForm, QuestionList, ConfirmModal, CategoryModal, MarkdownToolbar
- `backend/trpc/routes/admin/` — createLesson, updateLesson, deleteLesson, reorderLessons, createCategory, updateCategory
- `backend/trpc/routes/questions/` — upsert, delete, getByLessonId, updateCategoryByLesson
- No authentication gate — admin routes are unprotected
- No analytics, no bulk operations, no content staging

## Research Scope

Investigate and propose what the Admin panel should become. Consider:

1. **Content pipeline** — How should content creation flow? Drafts/staging/published? Bulk import? Translation workflow?
2. **Question management** — Better tooling for question banks, difficulty balancing, tagging, image support
3. **Analytics & insights** — What data would help content creators? Question difficulty stats, lesson engagement, weak spots across the student base
4. **Auth & access control** — Admin login, roles (editor/reviewer/admin), audit trails
5. **Category/curriculum design** — Tools for structuring the learning journey, reordering, prerequisites
6. **Media management** — Image uploads, asset organization
7. **What similar edtech admin panels do** — Look at patterns from learning platforms

## Deliverables

1. Research document at `board/active/WI-002-admin-panel-vision/research/findings.md`
2. Prioritized feature list with rationale
3. Proposed phasing — what to build first
4. Mail back to Hub with summary and recommendation

## Constraints

- Research only — no code changes
- Must be buildable with current stack (React Native/Expo, tRPC, Supabase)
- Admin panel serves Sammy as primary user, potentially other content editors later
- v1.1 Mobile polish (Phases 5-7) is running in parallel — Admin work should not conflict

---

*Hub — 2026-02-26*
