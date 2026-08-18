# PRD — Business Listing Directory & Internal CMS Platform

**Version:** 2.0
**Status:** Draft for review
**Stack:** Laravel (backend/CMS API) + Filament/Blade or Inertia (CMS UI) + Next.js (public frontend) + PostgreSQL + Dynamic RBAC (Spatie Laravel-Permission pattern)

---

## 1. Overview

A two-sided system with two **completely separate user pools**:

1. **Public Directory App** — Business Owners sign up/sign in, land on a fixed, simple dashboard. They can only: view their dashboard, manage (add/edit/delete) their own listings, view listing status, change password, and log out. No roles, no permissions — access is flat and identical for every business owner.
2. **Internal CMS/CRM** — Staff (Agents, Admins, and any future role) sign in separately. There is **no public sign-up path into the CMS** — Staff accounts are created internally only. Staff access is governed by a **Dynamic Role-Based Access Control (RBAC)** system: Admins can create new roles and assign granular permissions at runtime, without a code deployment.

Public landing pages are served at:
```
domain.com/{page_id}-{seo-slug}
e.g. domain.com/102931-asphalt-shingle-service-in-toronto
```
The numeric `page_id` is authoritative for routing; the slug is for SEO/readability only.

---

## 2. Goals

- Let business owners self-register and submit business info with minimal friction, with a deliberately narrow, fixed set of dashboard permissions.
- Keep the CMS completely closed to public sign-up — Staff accounts are provisioned internally only.
- Give Agents/Admins a clean pipeline to manage, call, and qualify Leads.
- Allow one-click conversion of a Lead into a Client.
- Allow manual Client creation (referrals, walk-ins) without a Lead.
- Let a Client have multiple landing Pages — one per Service × Area combination.
- Enforce consistent design: content is edited per structured field, but visual design/layout per template stays locked.
- Support **Dynamic RBAC**: Admins define roles and assign/revoke fine-grained permissions at runtime, and can grant/revoke individual permission overrides per Staff user.
- Maintain a full audit trail of status changes, edits, and permission/role changes.

---

## 3. User Pools & Access Model

### 3.1 Business Owners (Public Frontend)
Flat, fixed permission set — **not** governed by RBAC:

| Capability | Access |
|---|---|
| Sign up / Sign in | ✔ |
| View own dashboard | ✔ |
| Add / edit / delete own listings (while status allows) | ✔ |
| View listing status (read-only) | ✔ |
| Change password | ✔ |
| Log out | ✔ |
| Anything CMS-related | ✘ — no path exists |

### 3.2 Staff (Internal CMS) — Dynamic RBAC

- No public registration route exists for Staff. Accounts are created by an Admin inside the CMS only.
- Staff sign in through a separate, internal-only login.
- Access is controlled by **Roles**, which are collections of **Permissions**. Both are database-driven and manageable at runtime — not hardcoded enums.
- A default `Admin` role is seeded as a protected **system role** (cannot be deleted, always has full access) so the system can never be locked out of itself.
- A default `Agent` role is seeded with a sensible default permission set (Lead/Client/Page day-to-day operations) as a starting point — Admins can edit or clone it.
- Admins can create new roles (e.g. "Content Editor", "Senior Agent", "Read-only Auditor") and assign any combination of permissions to them.
- Optional per-user permission overrides (`grant` or `revoke`) let an Admin fine-tune one Staff member beyond their role, without creating a one-off role.

---

## 4. Core Entities (Summary)

- **BusinessOwner** — public account; flat access, no RBAC
- **StaffUser** — internal CMS account; access governed by RBAC
- **Role** — named, dynamic collection of permissions (e.g. Agent, Admin, Content Editor)
- **Permission** — a single granular action (e.g. `lead.edit`, `page.publish`, `role.manage`)
- **RolePermission** — pivot: which permissions belong to which role
- **StaffUserRole** — pivot: which role(s) a Staff user holds
- **StaffUserPermission** — optional direct grant/revoke override per Staff user
- **Lead** — a business submission from the public app, pre-qualification
- **Client** — a converted/qualified business, owns Pages
- **Page** — one landing page for a Client, tied to one Service + one Area + one Template
- **Template** — one of 3–4 fixed designs; defines the structured field schema
- **PageContent** — field values for a given Page, per its Template's schema
- **Niche** — taxonomy (Roofing, Plumbing, etc.)
- **Area** — taxonomy (Toronto, Mississauga, etc.)
- **ActivityLog** — audit trail across Lead/Client/Page/Role/Permission actions
- **Media** — uploaded images/assets, referenced by PageContent

Full field-level detail is in the accompanying ERD.

---

## 5. Functional Requirements

### 5.1 Public Directory App (Business Owner)
| ID | Requirement |
|---|---|
| FR1 | Business Owner can sign up / sign in (email + password, or OAuth — TBD). |
| FR2 | On first login, user is redirected to their Dashboard. |
| FR3 | Dashboard lists all businesses the user has submitted, with current status visible (e.g. "New", "In Review", "Booked"). |
| FR4 | User can add a new business listing (business name, niche, area, phone, email, description). |
| FR5 | User can edit their own listing while status is new/follow_up. Once converted, editing is locked — managed via CMS only. |
| FR6 | User can delete their own listing (soft delete). |
| FR7 | User can change their password and log out. |
| FR8 | Each listing submitted by a user creates one Lead record. |
| FR9 | No sign-up or access path exists from the public app into the CMS — the two are fully isolated. |

### 5.2 CMS Access & Authentication (Staff)
| ID | Requirement |
|---|---|
| FR10 | Staff sign in via a separate, internal-only login — no public self-registration route exists. |
| FR11 | Staff accounts are created only by an Admin, from within the CMS. |
| FR12 | Every CMS route/action is gated by a permission check — enforced server-side, not just hidden in the UI. |

### 5.3 Dynamic RBAC
| ID | Requirement |
|---|---|
| FR13 | Admin can create, edit, and delete Roles (except protected system roles). |
| FR14 | Admin can view a list of all Permissions, grouped by module (Lead, Client, Page, Template, Taxonomy, RBAC, Reporting). |
| FR15 | Admin can assign/revoke Permissions to/from a Role via a checkbox-style interface. |
| FR16 | Admin can assign one or more Roles to a Staff user. |
| FR17 | Admin can grant or revoke an individual Permission directly on a Staff user, overriding their Role(s). |
| FR18 | The `Admin` role is a protected system role — cannot be deleted, always retains full access, and at least one active Admin must exist at all times. |
| FR19 | All Role/Permission changes (create, edit, delete, assign, revoke) are recorded in ActivityLog with actor and before/after values. |

### 5.4 Lead Management
| ID | Requirement |
|---|---|
| FR20 | Staff with permission can view a Leads list with filters: status, niche, area, assigned agent, date submitted. |
| FR21 | Staff with permission can open a Lead and view/edit details, add call notes. |
| FR22 | Staff with permission can change Lead status: new → calling → no_answer → follow_up → booked → not_interested → converted. |
| FR23 | Every status change is recorded in ActivityLog (who, when, old→new status). |
| FR24 | Staff with permission can soft-delete (archive) a Lead — not a hard delete. |
| FR25 | System performs a duplicate check (business name + phone/email) before creating a new Lead, and flags likely duplicates. |
| FR26 | Staff with permission can click "Convert to Client" on a Lead → system creates a Client pre-filled from Lead data, sets Lead.status = converted, links converted_client_id, and redirects into the new Client profile. |

### 5.5 Client Management
| ID | Requirement |
|---|---|
| FR27 | Staff with permission can manually create a Client (no Lead required) — Client.source = manual. |
| FR28 | Client profile shows business info, assigned agent, status, and a list of all its Pages. |
| FR29 | Client can be edited or archived (soft delete). Archiving warns if it has published Pages. |

### 5.6 Page Management
| ID | Requirement |
|---|---|
| FR30 | From a Client profile, Staff with permission can create a new Page, selecting Service, Area, and one of the fixed Templates. |
| FR31 | Page editor shows a structured form matching the Template's field schema. Design is not editable — only field values. |
| FR32 | System auto-generates the SEO slug from Service + Area (editable override allowed). |
| FR33 | System auto-generates the numeric Page ID and final URL: domain.com/{page_id}-{slug}. |
| FR34 | Page status workflow: draft → pending_review → published → archived. |
| FR35 | Publishing a Page requires all required Template fields to be filled, and requires the `page.publish` permission. |
| FR36 | Each Page supports SEO meta fields: meta title, meta description, OG image. |
| FR37 | A Client can have unlimited Pages (multiple Services × multiple Areas). |

### 5.7 Templates
| ID | Requirement |
|---|---|
| FR38 | Staff with the `template.manage` permission manage a fixed library of 3–4 Templates, each with a defined structured field schema. |
| FR39 | Templates are reused across all Clients/Pages; editing a Template's design affects all Pages using it (not their content). |

### 5.8 Taxonomy
| ID | Requirement |
|---|---|
| FR40 | Staff with permission manage the Niche list (e.g., Roofing, Plumbing, HVAC). |
| FR41 | Staff with permission manage the Area list (e.g., Toronto, Mississauga). |
| FR42 | Lead, Client, and Page reference Niche/Area via foreign key (no free-text entry) for clean filtering/reporting. |

### 5.9 Activity Log
| ID | Requirement |
|---|---|
| FR43 | All create/update/delete/status-change/publish/role/permission actions are logged with actor, timestamp, and before/after values. |

### 5.10 Notifications (v1.1)
| ID | Requirement |
|---|---|
| FR44 | Agent(s) notified (email/Slack) when a new Lead is submitted. |
| FR45 | Admin notified when a Page is submitted for review. |

### 5.11 Media
| ID | Requirement |
|---|---|
| FR46 | Images are uploaded to a Media table/bucket and referenced by ID in PageContent, so assets are reusable across Pages. |

### 5.12 Basic Analytics (v1.1)
| ID | Requirement |
|---|---|
| FR47 | Track page views and form/CTA submissions per Page for basic performance reporting. |

---

## 6. Suggested Permission Set (seed data)

Grouped by module — Admins can extend this list at runtime:

- **Lead**: `lead.view`, `lead.create`, `lead.edit`, `lead.delete`, `lead.convert`, `lead.assign`
- **Client**: `client.view`, `client.create`, `client.edit`, `client.archive`
- **Page**: `page.view`, `page.create`, `page.edit`, `page.publish`, `page.archive`
- **Template**: `template.manage`
- **Taxonomy**: `taxonomy.manage`
- **RBAC**: `role.manage`, `permission.assign`, `staffuser.manage`
- **Reporting**: `reporting.view`

---

## 7. URL Routing Logic

1. Incoming request: `domain.com/102931-asphalt-shingle-service-in-toronto`
2. Router extracts leading numeric ID (`102931`) up to the first `-`.
3. Look up `Page` by `id = 102931`.
4. If not found or `status != published` → 404.
5. If found → load `Page.template_id` + `PageContent` for that page → render.
6. Slug portion after the ID is ignored for lookup (SEO-only); optionally 301-redirect to canonical slug if it doesn't match current Service/Area to avoid duplicate-content SEO issues.

---

## 8. Reporting Needs

- Lead funnel conversion rate (new → booked → converted) by agent, by niche, by source, by date range.
- Time-to-first-contact and time-to-conversion.
- Active Clients count, Pages published count.
- Staff activity — actions per Staff user, role/permission change history.
- (v1.1) Page-level traffic/conversion performance.

---

## 9. Open Questions

1. Auth method for Business Owner signup — email/password only, or also social login?
2. Should editing a Lead be locked for the Business Owner once an agent starts working it (status != new)?
3. Do we need a minimum required Page count / niche validation before a Client can be marked "active"?
4. Should Template design changes be versioned so existing published Pages aren't affected by a live template edit?
5. Should soft-deleted Leads/Clients/Pages be fully hidden from lists or shown in an "Archived" filter?
6. Can a Staff user hold multiple Roles simultaneously, or strictly one Role each? (ERD currently models many-to-many to support multiple.)
7. Should there be a minimum-permission floor that even a custom Role cannot go below (e.g. every Staff user can always view their own assigned Leads)?

---

## 10. Priority (v1 vs v1.1)

**v1 (must-have):** Business Owner signup/dashboard (flat access), Staff-only CMS login, Dynamic RBAC (Roles, Permissions, assignment, overrides), Lead CRUD + status pipeline, Lead→Client conversion, manual Client creation, Client profile, Page creation/editing with fixed Templates, Taxonomy (Niche/Area), ActivityLog, URL routing.

**v1.1 (fast-follow):** Notifications, basic Page analytics, duplicate-lead detection UI, Template versioning.

---

## 11. Technical Stack

| Layer | Choice | Notes |
|---|---|---|
| CMS Backend | **Laravel** | Eloquent ORM maps directly to ERD; migrations for schema; Policies/Gates for permission enforcement |
| RBAC | **Spatie laravel-permission** pattern | Roles/Permissions DB-backed, matches the Dynamic RBAC schema in this PRD |
| CMS Admin UI | **Filament** (or Inertia + Vue/React) | Rapid CRUD scaffolding for Leads/Clients/Pages/Roles/Permissions |
| Activity Logging | **Spatie laravel-activitylog** | Maps directly to the ActivityLog entity |
| Public Frontend | **Next.js** | SSR/SSG/ISR for SEO on public landing Pages; Laravel serves as pure API for this side |
| Database | **PostgreSQL** | Relational schema fits Lead/Client/Page/RBAC structure; JSONB for Template.field_schema |
| Auth (Business Owner) | Laravel Sanctum / Fortify (API) | Simple session or token auth for the public app |
| Auth (Staff) | Laravel session-based auth, separate guard from Business Owner | Enforces full isolation between the two user pools |
