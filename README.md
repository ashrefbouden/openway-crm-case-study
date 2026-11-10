# OpenWay Cons CRM — case study

**A production CRM for a study-abroad agency, built solo from first commit to production.**
Laravel 12 · React 19 + TypeScript · MySQL · Docker · CI with 649 automated tests

> Final-year engineering project (PFE, ESPRIT), defended November 2026 with high honors (mention *Très bien*).
> Built during a six-month internship at OpenWay Cons, an agency in Kairouan, Tunisia that helps students enrol in Italian universities and obtain their study visa.
> The source code is private (client data, agency business). This page describes the system; a walkthrough is available on request.

**Author:** Achref Bouden, full-stack developer · [LinkedIn](https://www.linkedin.com/in/achref-bouden-321842159) · [Email](mailto:ashrefbouden@gmail.com)

---

## The problem

The agency ran every student's case on spreadsheets, WhatsApp and paper folders. One case runs for months and goes through several dependent steps: language test, legalisation of diplomas, pre-enrolment on the Italian portal, university admission, then the visa. Each step has its own documents, deadlines and fees.

Three things kept going wrong:

1. **Nobody knew where a case really stood.** A step could be marked "done" while a document it needed was still missing.
2. **Money was hard to read.** Fees the agency earns were mixed with fees it only collects and forwards to third parties, so the totals overstated revenue.
3. **Clients had no visibility.** Every "where is my file?" question became a phone call or a WhatsApp message for an advisor.

## What I built

Two strictly separated portals on one API:

- **Back office** for the agency team (React + MUI): leads, candidate dossiers, the case pipeline, document review, finance, messaging, reporting.
- **Front office** for students (React + Tailwind): sign-up (email or Google), their own file and progress, document upload, messaging with their advisor, a map and catalogue of Italian universities, a free profile diagnostic, and an assistant that answers questions. Available in French, English, Italian and Arabic (right-to-left layout).

```
 React 19 SPA (TypeScript)
 ├─ Back office  /bo/*   MUI v7       staff token
 └─ Front office /fo/*   Tailwind 4   client token
            │  REST (JSON)
 Laravel 12 API (PHP 8.2)
 ├─ /api/backoffice/*    role-gated: employee · admin · superadmin
 ├─ /api/frontoffice/*   clients only, scoped to their own records
 └─ Route → Middleware → Controller → Service → Model → Resource
            │
 MySQL · file storage · queue worker · scheduler
 Docker images on GHCR · deploy and backup scripts
```

## Engineering decisions worth a look

**A pipeline that cannot lie.** The case pipeline is a dependency graph defined in config (7 phases: preparation, language test and legalisation in parallel, pre-enrolment, admission, visa, done). A phase's completion is never stored: it is recomputed from the underlying records on every read, so the status cannot drift from reality. Moving a case forward goes through one gate that refuses (HTTP 422) when dependencies are unmet. An admin can override, but only with a written reason that is recorded. Adding a phase is a config change, not a code change.

**Accounting that survives a dispute.** Each charge has a kind: agency fee (revenue) or pass-through (money collected for a third party, such as a visa centre fee). Dashboards only count real revenue. Payments are append-only: nothing edits or deletes one, and a mistake is corrected with a reversing entry, so both the error and the fix stay visible. Charges can be split into instalments with paid / pending / late states. Finance is restricted to the top role.

**An audit trail that never causes an outage.** Destructive and sensitive actions are logged with a snapshot of what was affected ("candidate X deleted" stays readable after the record is gone). The audit service never throws: if the log table fails, the business action still completes and the failure goes to the application log.

**Secrets handled like secrets.** Portal credentials stored for a case are encrypted at rest, never returned by list or detail endpoints, and only revealed through a dedicated endpoint that writes an access log entry.

**AI with guardrails, off by default.** Two assistants run on Groq's API:
- *Client assistant*: short answers grounded only in the agency's published information, the live university catalogue and the client's own (sanitised) file. It is told not to invent dates, prices or requirements, never to promise a visa, and to send the client to their advisor when the answer is not in its context. It detects Tunisian dialect (Derja) to reply in the client's language.
- *Advisor assistant*: summarises a client conversation before an advisor replies. Advisory only: nothing it produces writes to the dossier or affects the pipeline.

Before any text leaves the server, names become role labels and emails and long numbers become placeholders. A provider outage or rate limit degrades to "unavailable" instead of breaking the page.

**Security tested, not assumed.** A dedicated test suite checks the boundaries: a client token cannot reach the back office (enforced server-side, not by hiding routes in the SPA), one client cannot read another's application or documents, registration cannot set a role, and uploads reject executables and oversized files. Login and sensitive endpoints are throttled; suspended accounts are blocked at the middleware level.

## Other features

- Lead capture from the public site and a free profile diagnostic, and conversion of a lead into a client dossier
- Document workflow: upload (images are converted to PDF server-side), inline preview, verify or reject, expiry tracking
- Client-advisor messaging and in-app notifications, with notification analytics
- WhatsApp Business (Meta Cloud API) integration
- Interactive map and catalogue of Italian universities and programmes, filterable by level, language and Tunisian bac section; catalogue import and audit commands
- SEO for the public pages; generated OpenAPI documentation
- Operations commands: database backup, orphaned-file sweep, production config check, client reminders

## Quality and delivery

| | |
|---|---|
| Backend tests (PHPUnit, real MySQL) | 448 tests in 56 files |
| Frontend unit tests (Vitest) | 189 tests in 28 files |
| End-to-end tests (Playwright, full Docker stack) | 12 scenarios |
| Static checks in CI | TypeScript build, ESLint, Pint (PSR-12), i18n key parity across 4 languages |
| API routes | 210 |
| Domain | 35 models, 39 service classes, 67 migrations |
| Code size | ~25k lines of PHP (app), ~59k lines of TypeScript/TSX (frontend, excluding tests) |
| History | 323 commits, Feb–Oct 2026 |
| Releases | 24 release candidates (v1.0.0-rc1 to rc24, Sep–Oct 2026), in production |

Every push runs the backend suite on MySQL, the frontend tests and build, both Docker image builds, and the Playwright suite against the composed stack. A version tag builds and publishes the images to GitHub Container Registry; the server upgrades with one script, with backups and a post-deploy verification script.

## Stack

**Backend:** PHP 8.2, Laravel 12, Sanctum, Socialite (Google OAuth), MySQL, TCPDF, PHPUnit
**Frontend:** React 19, TypeScript 5.9, Vite 7, React Router, TanStack Query, react-hook-form + zod, MUI v7, Tailwind CSS 4, Leaflet, ApexCharts, i18next
**Ops:** Docker, Docker Compose, GitHub Actions, GitHub Container Registry, Playwright
**AI:** Groq API (OpenAI-compatible chat completions)

## What I learned

- Store facts, derive status. Every status I derived from records instead of storing stayed correct; every stored flag was a future bug.
- Treat money as a ledger, not as editable fields.
- Keep AI on the advisory side of the line. The rules that bind (phase gates, accounting, the audit trail) stay deterministic.
- Tests that cross the whole stack (two portals reading the same database row) catch what unit tests cannot.
