# Project Memory — Business Listing Platform

Pinned decisions & context for future chats. Read this first.

---

## Locked Decisions

- **Build order (5 milestones):** 1) Public Directory (frontend) → 2) Owner Dashboard (frontend) → 3) CRM backend (Laravel/Filament, Templates defined here) → 4) Landing Page Rendering (frontend, needs real Template schemas) → 5) Wire real API + integration. Reason: directory is read-only + sets the design system; dashboard reuses its components; landing render needs CRM templates first. See PHASES.md.
- **Stack:** Laravel (API + Filament CMS) + Next.js (public + owner) + PostgreSQL.
- **Frontend = business listing DIRECTORY (like citylocal101):** public browse/search listings by niche+area + basic listing detail page (`/business/{slug}`) + owner auth + owner dashboard + RENDERS landing pages. **NOT** the staff CMS.
- **Landing pages authored in CRM, NOT frontend:** agents build landing pages inside Filament (Templates) for CONVERTED clients only. Next.js only *renders* published ones at `/{id}-{slug}`. Frontend has no landing-page authoring UI.
- **Listing = Lead:** owner-submitted listing is the same record as the CRM Lead (public directory view + internal pipeline view). `Page` (landing page) is a separate entity created post-conversion.
- **URL spaces separate:** business listing detail = `/business/{slug}` (slug-addressed, NO numeric id); landing page = root `/{id}-{slug}` (numeric id, CRM-authored for converted clients). The `{id}-{slug}` id-prefixed form is reserved exclusively for landing pages. No collision.
- **Staff CMS:** Filament on Laravel `/admin`. Not Next.js.
- **Mocking:** typed data layer (`lib/api/`) + MSW mocks. TS types = the API contract, derived from ERD. Swap mock→real with zero component changes.
- **UI stack:** Tailwind + shadcn/ui (dashboard); custom Tailwind per Template (landing pages).
- **App Router**, TypeScript.

---

## RBAC

- **Spatie laravel-permission + custom revoke layer.**
- Roles/permissions/direct grants = pure Spatie. Bound to `staff` guard only.
- Extra table `staff_user_permission_revokes` — revoke always wins over inherited role perms.
- Effective perms = `(role perms ∪ direct grants) − revokes`.
- `Admin` = protected system role, bypasses revokes, always full access, ≥1 active Admin enforced.
- `Gate::before` hook honors revokes.

---

## Two-Pool Isolation (critical)

- `business_owners` (guard `sanctum`, flat access, public signup) vs `staff_users` (guard `staff`, RBAC, no public signup).
- Separate tables, models, guards. Spatie on `staff` guard only.
- Build & test this boundary before features.

---

## Schema Deviations from ERD (agreed)

1. `is_deleted` boolean → Laravel `SoftDeletes` (`deleted_at`). Show archived under filter.
2. Page public ID → high-start auto-increment sequence (start `100000`).
3. `Lead.source` vs `Client.source` enum naming normalized when writing migrations.

---

## Open Questions — Resolved (v1)

1. Owner auth = email/password only (no OAuth v1).
2. Owner Lead edit locked once `status ∉ {new, follow_up}`.
3. No min Page/niche rule for Client "active" — manual staff toggle.
4. Template versioning = deferred to v1.1.
5. Soft-deleted records shown under "Archived" filter, not hidden.
6. Staff can hold MULTIPLE roles (many-to-many). Effective = union.
7. No min-permission floor v1.

---

## v1 vs v1.1

- **v1:** Owner signup/dashboard, staff CMS login, Dynamic RBAC, Lead CRUD+pipeline, Lead→Client convert, manual Client, Client profile, Page CRUD w/ fixed Templates, Taxonomy, ActivityLog, URL routing.
- **v1.1:** Notifications, page analytics, duplicate-lead UI, Template versioning.

---

## Source of Truth

- `PRD.md` — requirements (FR1–FR47).
- `ERD.mermaid` — full field-level schema.
- `ARCHITECTURE.md` — topology + directory structure.
- `PHASES.md` — execution plan.
