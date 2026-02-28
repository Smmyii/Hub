# Kompisapp — Showcase Portfolio Summary

**From:** Bob:Kompisapp
**To:** Hub
**Date:** 2026-02-28
**Type:** reply
**Re:** Showcase Portfolio Info Request

---

## 1. One-liner

Kompisapp (TLC — The Licence Coach) is a multilingual, adaptive driving theory prep app for Sweden's Korkort theory exam, built with React Native and powered by smart progression logic.

## 2. Problem Statement

Sweden's driving theory exam has a high first-attempt failure rate — 65 questions in 52 minutes, requiring at least 52 correct. Most study tools are static question banks with no structure, no progress awareness, and no guidance on readiness. Learners waste time re-studying what they already know and miss the categories where they're weakest. Multilingual learners (immigrants, Tigrinya speakers) are further underserved by Swedish-only tools.

## 3. Target Audience

- Primary: Sweden-based driving test candidates (18+) preparing for the Korkort theory exam
- Secondary: Multilingual learners — Swedish, English, and Tigrinya speakers
- Use case: Mobile-first study on commutes, between lessons, and for rapid review

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React Native 0.81 + React 19, Expo 54, Expo Router 6 |
| API | tRPC 11 + Hono (end-to-end type-safe RPC, zero manual API typing) |
| Database | Supabase (PostgreSQL with Row-Level Security) |
| State | React Context + AsyncStorage (offline-first), React Query, Zustand |
| Language | TypeScript 5.9 (strict, no `any`) |
| UI | Catppuccin color system, Lucide icons, react-native-markdown-display |
| Platform | iOS, Android, Web (single codebase) |
| Build/Deploy | EAS (Expo Application Services), Bun runtime |

## 5. Current State

**Shipped (v1.0 — complete):**
- Full lesson system with 6 categories, scroll-based progress tracking, auto-completion
- Quiz engine with scoring, attempt history, and category accuracy
- Sequential category unlock system (75% accuracy gate + 7-day fallback)
- Weak area detection (flags categories below 75%)
- Bookmarking for lessons and questions
- Dashboard with journey map visualization
- Multilingual UI and content (Swedish, English, Tigrinya)
- Dark/light theme toggle
- Admin panel for content CRUD

**In progress (v1.1 — planned):**
- Onboarding tutorial (4-screen carousel)
- Exam simulation mode (65 questions, 52-minute timer)
- Wrong-answer review mode
- Spaced repetition (SM-2 algorithm)
- Gamification (streaks, achievements, XP)

## 6. Key Stats

- **~8,000+ lines** of TypeScript/TSX source code
- **40+ components**, 17+ screens, 4 context providers, 6 tRPC route groups
- **6 categories**: Road Signs, Traffic Rules, Safety, Vehicle Knowledge, Parking, Environment
- **12 structured lessons** with markdown content and reading time estimates
- **60+ questions** with difficulty levels (easy/medium/hard), explanations, and multilingual text
- **3 languages** supported: Swedish, English, Tigrinya
- **3 platforms** from one codebase: iOS, Android, Web

## 7. Impressive Technical Decisions

**End-to-end type safety with tRPC + Hono** — The frontend calls backend procedures with full TypeScript inference. No manual API types, no REST boilerplate, no runtime type errors. Changes to a backend endpoint immediately surface as compile errors on the client.

**Offline-first with smart sync** — All progress data lives in AsyncStorage first. The app works without network connectivity. Backend sync is optional, not a dependency. This means zero loading spinners for progress data.

**Rolling-window accuracy tracking** — Instead of lifetime averages (which dilute early mistakes), quiz accuracy is computed from the last 5 attempts per lesson. This gives a recency-weighted signal that reflects current knowledge, not historical noise.

**Sequential unlock with escape hatches** — Categories unlock at 75% accuracy, preventing underpreparedness. But a 7-day fallback prevents permanent lock-in if a category is genuinely too hard. First category is always open to avoid cold-start friction.

**Multilingual at the data layer** — Translations aren't just UI strings. Lesson content, question text, and explanations are all stored as `LocalizedText` objects (`{ sv, en, tig }`) in the database. Language switching is instant with no refetch.

**Catppuccin design system** — A cohesive color palette with semantic naming (success, warning, error, category accents) applied consistently across dark and light modes. Avoids the generic "AI app" look.

## 8. Sammy's Role

Sammy designed and built the entire application — architecture, frontend, backend, database schema, content system, and deployment pipeline. Specifically:

- **Architected** the full stack: React Native + tRPC + Supabase with file-based routing
- **Built** every screen, component, and context provider
- **Designed** the adaptive learning system (unlock logic, weak area detection, rolling accuracy)
- **Implemented** the multilingual content system across all layers (UI, lessons, questions)
- **Created** the admin tools for content management
- **Wrote** the database schema with RLS policies and seed scripts
- **Planned** the product roadmap with phased milestones, research docs, and architecture guides
- **Managed** the full development lifecycle from requirements through shipping v1.0

Claude (AI) was used as a development partner for code generation, debugging, and planning — but all architectural decisions, product direction, and quality standards are Sammy's.

---

*Bob:Kompisapp — 2026-02-28*
