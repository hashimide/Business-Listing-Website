# Execution Phases — Business Listing Platform

5 product milestones. Frontend-first (mock-driven), then CRM backend, then wire real API.
Dense checklist. Finish + review each milestone before the next.

**Build order:** Directory → Owner Dashboard → CRM (Laravel/Filament, incl. Templates) → Landing Page Rendering → Integration.
Reason: directory is read-only + establishes the design system; owner dashboard reuses its components; landing-page rendering needs real Template schemas from the CRM, so it comes after.

---

## PHASE 0 — Foundation & Contract  ✅ DONE
- [x] Init Next.js 16 (App Router, TS, Tailwind v4, shadcn/ui).
- [x] `lib/api/types.ts` — TS contract for frontend entities (BusinessOwner, PublicListing/OwnerListing = Lead, LandingPage, Template, Niche, Area, Media, auth).
- [x] Enums as TS union types (LeadStatus, PageStatus, source, etc.).
- [x] `lib/api/client.ts` — fetch wrapper (base URL, bearer token, typed errors).
- [x] MSW setup (browser + node) + `lib/api/mock/` handlers + seed data.
- [x] Per-entity api modules: listings, landing, taxonomy, auth.
- [x] `proxy.ts` (Next 16, renamed from middleware) — guards `/dashboard`, `/settings`.
- [x] Verified: typecheck clean + production build passes.

---

## PHASE 1 — Public Directory (citylocal101-style)  ✅ DONE
Frontend. Read-only. Establishes design system + reusable components.
- [x] Design system pass: brand-blue primary/accent tokens in `globals.css` (light+dark), shadcn base-nova theme.
- [x] Site shell: header (logo, compact GET search, Browse/Owner-Login) + footer, wired in `layout.tsx`.
- [x] `app/page.tsx` — directory home: hero search, browse by niche, browse by area, recent listings. (`force-dynamic` in mock era; static/ISR revisited Phase 5.)
- [x] `components/directory/` — listing card, `SearchFilters` bar; UI primitives added: card, badge, input/select.
- [x] `app/search/page.tsx` — results by q + niche_id + area_id (native GET form, no client JS), pagination + empty state + `loading.tsx`.
- [x] `app/business/[slug]/page.tsx` — listing detail at **`/business/{slug}`** (slug-addressed, resolved by slug; NO numeric id — the `{id}-{slug}` form is reserved for CRM landing pages), async params, `PageProps<>` helper.
- [x] `notFound()` for missing slug. Listing = Lead record (public fields only). SEO: `metadataBase`, per-page title/description, `rel=canonical` (`/business/{slug}`), explicit `noindex` on 404.
- [x] Responsive, semantic HTML, loading/empty states via data layer.

> **Next 16 note:** listing fetch suspends → response streams → `notFound()` is a *soft* 404 (200 + noindex), not a hard 404 status. SEO-safe via explicit `noindex`. Hard status would need a `proxy.ts` check before the body streams. Verified via `next build` + `next start`.

---

## PHASE 2 — Owner Dashboard (auth + listing CRUD)  ← NEXT
Frontend. Reuses Phase 1 components (cards, niche/area selectors, badges).
- [ ] Signup page (email + password only). Login page.
- [ ] Session: token in httpOnly-style cookie (`owner_token`); `proxy.ts` guard active.
- [ ] `app/(owner)/dashboard/page.tsx` — owner's listings + status badges (new/in_review/booked/converted).
- [ ] Add listing form (business_name, niche, area, phone, email, description) → creates Lead.
- [ ] Edit listing — locked when not editable (status ∉ {new, follow_up}).
- [ ] Delete (soft) listing. Niche/Area via FK-backed dropdowns.
- [ ] Settings: change password, logout.
- [ ] Empty/loading/error states. Mock auth until Sanctum (Phase 5).

---

## PHASE 3 — CRM Backend (Laravel + Filament)
The internal system. Heavy backend lift. Templates defined HERE.

**3a. Scaffold + isolation**
- [ ] Fresh Laravel + PostgreSQL. Install spatie/laravel-permission, spatie/laravel-activitylog, sanctum, filament.
- [ ] `config/auth.php` — two guards (`sanctum` owner, `staff`). Models: BusinessOwner, StaffUser (+HasRoles on staff guard).
- [ ] Prove isolation: owner token cannot hit staff routes; Filament `/admin` = staff only.

**3b. Dynamic RBAC**
- [ ] Migrations: Spatie tables + `staff_user_permission_revokes`.
- [ ] Seed permissions (Lead/Client/Page/Template/Taxonomy/RBAC/Reporting per PRD §6).
- [ ] Seed Admin (system, protected) + Agent roles. `Gate::before` revoke-override; Admin bypasses.
- [ ] Enforce ≥1 active Admin invariant. ActivityLog on all role/permission changes (before/after).

**3c. Domain migrations & models**
- [ ] Migrations: Lead, LeadNote, Client, Page, Template, PageContent, Media, Niche, Area, ActivityLog.
- [ ] `SoftDeletes` on Lead/Client/Page. `pages_id_seq` start 100000. `field_schema` JSONB.
- [ ] Eloquent relationships wired. Factories + seeders matching frontend mock shapes.

**3d. Filament CMS**
- [ ] Staff panel `/admin`, permission-gated resources (Policies).
- [ ] Resources: Lead, Client, Page, Template, Niche, Area, Media, Role, Permission, StaffUser.
- [ ] Lead list filters (status/niche/area/agent/date) + detail + call notes + status pipeline.
- [ ] "Convert to Client" action (LeadConversionService). Manual Client create. Client profile w/ Pages.
- [ ] **Templates**: manage 3–4 fixed Templates w/ structured `field_schema`.
- [ ] Page editor — structured form from Template schema, design locked. Publish gate: required fields + `page.publish`.

---

## PHASE 4 — Landing Page Rendering (frontend, consumes real Templates)
Frontend. Built AFTER CRM defines real Template schemas.
- [ ] `app/[pageSlug]/page.tsx` — parse `{id}-{slug}`, fetch Page + PageContent by ID.
- [ ] 404 if not found / not published. 301 to canonical slug on mismatch.
- [ ] `components/landing/` — one render component per Template, driven by field_schema.
- [ ] SEO meta: title, description, OG image. ISR config.
- [ ] NOTE: render only — authoring lives in Filament (Phase 3d).

---

## PHASE 5 — Wire Real API + Integration
Connect frontend to Laravel; swap off mocks.
- [ ] API endpoints: owner auth (Sanctum), owner listings CRUD (creates Leads), public listings/search, listing detail, published page, taxonomy.
- [ ] Swap frontend `lib/api/mock/` → real client (set NEXT_PUBLIC_API_URL, drop MSW). Verify zero component changes.
- [ ] End-to-end: owner signup → submit listing → appears as Lead in CRM → agent converts → Client → Page built → published → live `/{id}-{slug}`.
- [ ] Edge cases: bad ID, unpublished, slug mismatch. Server-side permission enforcement verified.
- [ ] ActivityLog coverage audit (FR43). Duplicate-lead check (business_name + phone/email).

---

## PHASE 6 — v1.1 Fast-Follow
- [ ] Notifications (email/Slack): new Lead → agents, Page review → admin.
- [ ] Page analytics: views + CTA submissions.
- [ ] Duplicate-lead detection UI.
- [ ] Template versioning (design changes don't affect published pages).

---

**Rule:** finish + review each milestone before the next. Frontend milestones (1–2) locked before CRM (3) starts; Landing rendering (4) needs Templates from (3).
