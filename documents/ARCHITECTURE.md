# Architecture — Business Listing Directory & Internal CMS

**Version:** 1.0
**Approach:** Frontend-first (Next.js) → Backend (Laravel). Contract-locked via shared TypeScript types.

---

## 1. System Topology

Three deployables, one database.

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│      Next.js (Frontend)     │     │      Laravel (Backend)       │
│   = Business Listing        │     │                              │
│     Directory (citylocal101)│◄───►│  • REST/JSON API (Sanctum)   │
│                             │ API │  • Filament CMS  /admin       │
│  • Public directory + search│     │    (Leads/Clients/Pages/RBAC)│
│  • Listing detail pages     │     │  • Landing pages AUTHORED    │
│  • Owner auth + dashboard   │     │    here (Templates)          │
│  • RENDERS landing pages    │     │  • Dynamic RBAC (Spatie+)    │
└─────────────────────────────┘     └──────────────┬───────────────┘
                                                    │
                                            ┌───────▼───────┐
                                            │  PostgreSQL   │
                                            │  (JSONB for   │
                                            │  template     │
                                            │  schemas)     │
                                            └───────────────┘
```

| Surface | Tech | Auth | URL | Notes |
|---|---|---|---|---|
| Public directory + search | Next.js (SSR/ISR) | none | `/`, `/search` | browse listings by niche+area |
| Listing detail page | Next.js (ISR) | none | `/business/{slug}` | basic public page for a submitted listing (slug-addressed; id-prefixed URLs are landing-page only) |
| Landing page | Next.js (SSG/ISR) | none | `/{id}-{slug}` | rich templated SEO page; **authored in CRM**, converted clients only |
| Owner dashboard | Next.js | Sanctum token | `/dashboard` | signup/login, manage own listing |
| Staff CMS | Filament (Laravel) | staff session guard | `/admin` | Leads/Clients/Pages/RBAC + **builds landing pages** |
| API | Laravel | Sanctum + guards | — | serves Next.js only |

**Key distinction:** Frontend has **no landing-page authoring UI**. Agents build landing pages inside Filament; Next.js only renders published ones. Directory listings (owner-submitted) and landing pages (agent-built) are separate URL spaces — listing detail is prefixed `/business/…` so it never collides with the root `/{id}-{slug}` landing-page route.

---

## 2. Two-Pool Isolation (Security Spine)

Two completely separate user pools. Never merged.

| Concern | Business Owner | Staff |
|---|---|---|
| Table | `business_owners` | `staff_users` |
| Model | `App\Models\BusinessOwner` | `App\Models\StaffUser` |
| Guard | `sanctum` (API token) | `staff` (session) |
| Signup route | public | **none — Admin-created only** |
| RBAC | ❌ flat access | ✅ Spatie, `guard_name=staff` |
| Filament `/admin` | ❌ never | ✅ |

Spatie permissions bound to `staff` guard only → an Owner token can never resolve a permission check.

---

## 3. Frontend Directory Structure (Next.js — App Router)

```
frontend/
├── app/
│   ├── page.tsx                  # DIRECTORY HOME — browse/search listings
│   ├── search/page.tsx           # search results (niche + area filters)
│   ├── business/
│   │   └── [slug]/page.tsx       # LISTING DETAIL — /business/{slug}, ISR
│   ├── [pageSlug]/
│   │   └── page.tsx              # LANDING PAGE — /{id}-{slug}, CRM-authored, ISR, 404 if unpublished
│   ├── (owner)/                  # route group — owner-only, guarded by middleware
│   │   ├── login/page.tsx
│   │   ├── signup/page.tsx
│   │   ├── dashboard/
│   │   │   ├── page.tsx          # owner's own listing(s) + status
│   │   │   └── listings/
│   │   │       ├── new/page.tsx
│   │   │       └── [id]/edit/page.tsx
│   │   └── settings/page.tsx     # change password
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── ui/                       # shadcn/ui primitives
│   ├── directory/                # listing cards, search bar, filters
│   ├── owner/                    # dashboard-specific components
│   └── landing/                  # template-rendering components (1 per Template)
├── lib/
│   ├── api/
│   │   ├── types.ts              # ⭐ THE CONTRACT — TS types from ERD
│   │   ├── client.ts             # fetch wrapper (base URL, auth header)
│   │   ├── listings.ts           # directory + owner listing CRUD (= Lead on backend)
│   │   ├── landing.ts            # fetch published landing page + content by ID
│   │   ├── taxonomy.ts           # niche/area lists
│   │   ├── auth.ts               # owner signup/login/logout/password
│   │   └── mock/                 # MSW handlers + mock JSON
│   ├── auth/                     # owner session helpers
│   └── utils.ts
├── proxy.ts                      # Next.js 16 (renamed from middleware.ts) — guards (owner)/* routes
├── instrumentation.ts            # starts server-side MSW mocking (dev)
└── package.json                  # Tailwind v4 = CSS-config (no tailwind.config.ts)
```

**Contract rule:** every network read/write goes through `lib/api/*`. Components never call `fetch` directly. Swap `mock/` → real API with zero component changes.

---

## 4. Backend Directory Structure (Laravel — later phase)

```
backend/
├── app/
│   ├── Models/
│   │   ├── BusinessOwner.php     # sanctum guard
│   │   ├── StaffUser.php         # staff guard + Spatie HasRoles
│   │   ├── Lead.php  Client.php  Page.php  Template.php
│   │   ├── PageContent.php  Media.php  Niche.php  Area.php
│   │   └── LeadNote.php
│   ├── Filament/                 # CMS resources (staff panel)
│   │   ├── Resources/            # LeadResource, ClientResource, PageResource, RoleResource...
│   │   └── Pages/
│   ├── Http/
│   │   ├── Controllers/Api/      # endpoints for Next.js
│   │   └── Middleware/
│   ├── Policies/                 # permission enforcement
│   ├── Services/
│   │   ├── LeadConversionService.php   # Lead → Client
│   │   └── Rbac/RevokeGate.php          # revoke-override layer
│   └── Providers/
│       ├── AuthServiceProvider.php      # Gate::before revoke hook
│       └── Filament/AdminPanelProvider.php
├── database/
│   ├── migrations/
│   └── seeders/                  # permissions, Admin/Agent roles, templates
├── config/auth.php               # two guards
└── routes/api.php
```

---

## 5. Key Technical Decisions

- **Listing = Lead (one record, two views):** what an owner submits is a public **directory listing** AND the internal **Lead** in the CRM pipeline — same row. Public directory shows it (basic fields); CRM works it (status pipeline, notes, assignment). A `Page` (landing page) is a *separate* entity, created later by an agent only after conversion.
- **RBAC:** Spatie laravel-permission for roles/permissions/direct grants + custom `staff_user_permission_revokes` table. Effective perms = `(role ∪ direct grants) − revokes`. Admin system role bypasses revokes.
- **Soft deletes:** Laravel `SoftDeletes` (`deleted_at`), **not** ERD's `is_deleted` boolean. Archived shown under filter.
- **Page public ID:** auto-increment sequence starting at `100000` (`ALTER SEQUENCE pages_id_seq RESTART WITH 100000`).
- **Template schema:** `field_schema` as Postgres **JSONB**. PageContent = EAV keyed to it. Design locked, only values editable.
- **Activity log:** Spatie laravel-activitylog + explicit logging for role/permission before/after (FR19).
- **Routing:** `/{id}-{slug}` → extract leading numeric ID → lookup Page → 404 if not published → 301 to canonical slug if mismatch.

---

## 6. API Contract Flow (frontend-first)

```
ERD  →  lib/api/types.ts (TS types)  →  frontend built against types
                    │
                    └──►  Laravel API later satisfies same shapes  →  swap mock for real
```

The ERD is the single source of truth. Types are derived from it. Backend conforms to types. No reshaping.
