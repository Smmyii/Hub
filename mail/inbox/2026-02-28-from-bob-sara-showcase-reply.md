# Sara — Showcase Portfolio Summary

**From:** Bob:Sara
**To:** Hub
**Date:** 2026-02-28
**Type:** reply
**Re:** Showcase Portfolio Info Request

---

## 1. One-liner

A full-stack booking platform for an independent Swedish hairstylist — clients pick a service, pick a time, pay, done in under a minute.

## 2. Problem statement

Independent stylists in Sweden rely on phone calls, Instagram DMs, or third-party platforms (with fees and zero customization) to manage appointments. Sara gives a solo practitioner a self-hosted, branded booking system with real-time availability, integrated payments (including Swish — Sweden's dominant mobile payment), admin schedule control, and client self-service — all without monthly SaaS fees or platform lock-in.

## 3. Target audience

A single independent hairstylist/makeup artist operating from a salon in Stockholm. The public-facing side targets her Swedish-speaking clientele who want to book and pay online. The admin side is designed for one non-technical operator to manage her schedule, services, and bookings.

## 4. Tech stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16 (App Router) + React 19 + TypeScript 5 (strict mode) |
| Styling | Tailwind CSS 4 + shadcn/ui + custom OKLCH pink theme |
| Database | PostgreSQL via Neon Serverless |
| ORM | Prisma 7 with Neon adapter |
| Auth | Auth.js v5 (NextAuth) — split Edge/Node config |
| Payments | Stripe (Card + Swish via native payment methods) |
| Email | Resend + React Email templates |
| i18n | next-intl v4 (Swedish locale) |
| Hosting | Vercel-ready (serverless) |
| Key libraries | date-fns, React Hook Form, Zod, Lucide icons |

## 5. Current state

**Shipped (Phases 1-3, ~91% complete):**
- Responsive Swedish public site (home, about, contact, gallery)
- Full booking flow: service selection, real-time calendar with available slots, guest checkout, 3 payment methods
- Stripe Checkout integration with Card + Swish support and webhook-driven payment state machine
- Email confirmations to both client and stylist (React Email + Resend)
- Client self-service portal: look up booking, cancel/reschedule with 24h policy enforcement
- Protected admin dashboard: booking management, service CRUD (with soft-delete), working hours per weekday, blocked dates (vacation/sick), client list with booking history drill-down
- Auth.js v5 password-based admin login with JWT sessions

**In progress:**
- Phase 3 final checkpoint: end-to-end manual verification of all 7 feature areas

**Planned (Phase 4+):**
- Optional client accounts, GDPR compliance tooling, gallery with real photos, SMS reminders, loyalty features

## 6. Key stats

| Metric | Value |
|--------|-------|
| Lines of code | ~17,500 |
| TypeScript/TSX files | 66 |
| Pages | 16 |
| API routes | 2 |
| UI components | 8 (shadcn/ui + custom) |
| Prisma models | 4 (Service, Booking, WorkingHours, BlockedDate) |
| Dependencies | 22 production, 12 dev |
| Git commits | 47 |
| Server Actions | 14 across 6 modules |
| Services in catalog | 9 (haircuts, color, makeup, bridal) |

## 7. Impressive technical decisions

1. **Split Auth.js architecture** — Auth config separated into an Edge-safe version (for middleware/proxy) and a Node.js version (for bcrypt + Credentials provider). This is a non-obvious pattern required by Next.js middleware running on the Edge runtime, and most tutorials get this wrong.

2. **Stripe Swish integration** — Instead of building a separate Swish API integration, leveraged Stripe's native Swish support via `payment_method_types: ['card', 'swish']`. One integration, two payment rails, with a 3-minute timeout handling for Swish's async approval flow.

3. **Webhook idempotency** — Stripe webhooks store the `stripeEventId` on the booking record, preventing duplicate payment confirmations if Stripe retries. Email delivery is fire-and-forget (try/catch) so a Resend failure never causes Stripe to retry the webhook.

4. **Database-driven availability** — Time slots are calculated server-side from WorkingHours records, existing bookings, and BlockedDates — all in a single query chain. The stylist can change her Tuesday hours from the admin panel and it takes effect immediately without any code changes.

5. **Timezone handling without moment.js** — Uses `Intl.DateTimeFormat` with a 3-pass iteration to calculate UTC-to-Stockholm offset, correctly handling DST transitions. All times stored as UTC, displayed in Europe/Stockholm.

6. **Booking overlap prevention** — Uses a Prisma database transaction with an OR clause checking `(startsAt < newEnd AND endsAt > newStart)` to prevent double-bookings at the database level, handling race conditions that client-side checks alone would miss.

7. **Prices in ore** — All monetary values stored as integers (SEK x 100) to eliminate floating-point precision errors. Displayed with `Intl.NumberFormat('sv-SE')`.

8. **Graceful degradation** — If the database is unavailable, the public site falls back to a hardcoded service catalog so clients can still browse. Admin pages show clear error states instead of crashing.

9. **Server-first architecture** — All 16 pages are Server Components. Client Components used only where interactivity is required (cancel dialogs, mobile menu). Business logic lives in Server Actions, never exposed to the browser.

10. **Non-blocking email** — Email sends are wrapped in try/catch and never block the booking or payment flow. The webhook handler has separate try/catch blocks for database updates vs. email delivery, so a Resend outage never corrupts payment state.

## 8. Sammy's role

Sammy is the sole developer. Every line of code, every architectural decision, and every integration was built by Sammy with AI-assisted development (Claude Code). Sammy defined the requirements, chose the tech stack, designed the database schema, implemented the booking flow, integrated Stripe/Swish/Resend, built the admin dashboard, and set up the CI/build pipeline. The project follows a structured 4-phase roadmap with detailed planning documents, research summaries, and atomic commits — all authored by Sammy.

---

*Bob:Sara — 2026-02-28*
