# Project Selections Portal — Architecture & Delivery Plan

Status: **Draft v0.2** (2026-09-28): revised after the Aligned reference analysis · Owner: Digital / Project Sales

## 1. Purpose

A private, invite-only portal where hotel brands, architects/designers and
developers work with our Project Sales and Marketing staff on specific
hospitality projects. It is a place for **selections and decisions**:

- each project has its own area, reached by a clean URL
  (`portal.metrofloors.com/harbor-pine-austin`)
- a **project micro-catalog**: the products selected or under consideration,
  shown by area (Guest Rooms, Corridors, Lobby, …) with status, specs,
  documents and stakeholder reviews
- product review and approval by each stakeholder
- sample requests
- spec, document and submittal libraries (drawings, specs, submittals)
- quotes
- message threads per project with a named rep, with file sharing
- a **shared action plan**: stages → steps with owner, due date and status, some
  steps internal-only
- a welcome from the named rep and a **stakeholder list** on both sides
- **staff-only** space per project: Salesforce opportunity fields, handoff
  notes, internal steps, internal notes on threads, and client **engagement**
  (who visited, what they looked at)

Meeting scheduling stays in email. Projects are registered internally (staff
create them); customers never self-register.

This replaces the equivalent features of the hosted TeamAligned portal
(`metrofloors.teamaligned.com`), with the data under our own control.

## 2. Guiding decisions

| # | Decision | Why |
|---|----------|-----|
| D1 | **Separate app beside Adobe Commerce**, not a Magento module | Porting away from Commerce later must not mean rewriting the portal. The portal talks to Commerce only over its public APIs. |
| D2 | **Laravel 11 / PHP 8.3** | PHP is already on the box and the in-house team already knows it from Magento. Laravel provides auth, policies, queues, mail and storage out of the box. C# would add a second runtime to the shared server. |
| D3 | **The portal owns its own data** (projects, selections, reviews, messages, documents) in its own database schema | This is what makes the portal portable. Commerce is read-only for catalog data, and Salesforce/M3 are integrations, not stores. |
| D4 | **Identity is pluggable** behind an `IdentityProvider` interface; users are keyed by an internal UUID with a separate `identity_links` table | We can move customers from Commerce logins to another provider without touching project data. |
| D5 | **Staff sign in with Entra ID** (OIDC). Duo MFA and conditional access come from the existing Entra policies | There is no second staff password, and staff offboarding happens in one place. |
| D6 | **Customers sign in with Adobe Commerce customer accounts** in a dedicated customer group, **Project Portal** | This is the brief. We reuse Commerce's credential store, password reset and lockout. See §4.3 for the trade-off. |
| D7 | **Product data is snapshotted** into the selection when a product is added, then refreshed on a schedule | A project's record of what was selected survives catalog changes, discontinued SKUs and a platform move. |

| D8 | **Visibility is filtered on the server.** A single `ProjectPresenter::forViewer()` decides what each role receives, and internal steps, notes, tabs and CRM fields are never sent to a client browser | Aligned's analysis flags this as the main risk in a room product. Staff get a **Client preview** toggle that renders the same filtered output. |
| D9 | **Invite-only, no anonymous links** | This is a deliberate difference from Aligned, which allows open links, name and email gates, and anonymous visitor tracking. Our users are named project stakeholders, and every view is tied to a real person. |
| D10 | **Fixed, domain-specific tabs** (Overview, Plan, Selections, Samples, Documents, Quotes, Messages, Team + staff-only Internal, Engagement). **Not** Aligned's free-form tab and section builder | Our value is the selections workflow, not a page builder. Only the Overview holds rep-authored content: a welcome and pinned documents. A section registry can come later if reps ask for it. |

## 2a. What the Aligned analysis changed

The reference pack (`docs/reference/aligned/`) shows that Aligned is a
general-purpose "digital sales room" product. It has **no product selections,
samples or quotes**, so the micro-catalog is ours to design. It does have
collaboration patterns this plan was missing:

| Aligned feature | Our decision | Phase |
|---|---|---|
| Mutual Action Plan (stages → steps, owner, date, status: Not started / On track / At risk / Delayed, internal flag, % counted on client-visible steps) | **Adopt.** Replaces the static milestone list. | 1 |
| Internal tab (CRM fields, internal tasks, handoff notes) | **Adopt** as a staff-only *Internal* tab. CRM fields are read from Salesforce. | 1 (notes), 2 (Salesforce) |
| Comments as *public* or *internal note* | **Adopt** on message threads and plan steps. | 1 |
| Edit ⇄ Preview (buyer view strips internal content) | **Adopt** as *Staff view / Client preview*, filtered on the server (D8). | 1 |
| Welcome card from the owner + Stakeholders (both sides) | **Adopt** on Overview. | 1 |
| Dual logos + banner per room | **Adopt.** Our mark + client mark and a per-project colour and banner. | 1 |
| Engagement analytics (visits, time per tab, most-viewed content, trend) | **Adopt, simplified.** Staff-only *Engagement* tab fed by an append-only event table. | 1 (events), 2 (dashboard) |
| Notification matrix (Email / Slack / Teams) | **Adapt.** Customers get **email** (immediate or daily digest). Staff get **Salesforce notifications** (bell + Salesforce mobile), with email as the fallback. No Slack or Teams. | 1 (email), 2 (Salesforce) |
| Templates (room saved for reuse) | **Adapt** as *brand templates*: areas, plan stages, welcome text and pinned brand-standard documents. Launch set: **Best Western, Marriott, Global Hotel Alliance** (§6a). | 1 |
| Content library (folders, labels, viewer) | **Adapt.** Back the company library with **SharePoint** via Microsoft Graph, since Marketing already keeps content there. Projects pin items from it. | 2 |
| Deep-linkable panels (`?actionItemId=`, `?sectionId=`) | **Adopt** for selections, steps and threads. | 1 |
| Share link access modes, anonymous visitor tracking | **Skip** (D9). | – |
| Free-form tab and section builder, embeds (Loom, Miro, PandaDoc…) | **Skip** for now (D10). | – |
| AI buyer assistant, seller agent, deal builder, deal insights | **Defer.** A project assistant grounded in selections, specs and threads is a good phase-3 candidate. | 3+ |
| Stakeholder org chart, cross-room task manager, manager dashboard | **Defer.** A staff "My open steps" view is a cheap first step. | 3 |

Tech lessons we're taking from it:
- Pick **one** UI system. Aligned mixes three: CSS Modules, Tailwind and Bootstrap.
- Opening a task in Aligned writes read-tracking. Decide explicitly which reads
  count as "seen".
- Model plan steps in their own tables, because they're queried across
  projects. Store the few free-form bits as validated JSON.

## 2b. Experience & theming

The portal is a **design-forward selection space**, not an admin tool. Every
customer area should look like that chain's or developer's space and like
Metro's at the same time.

### Three theme layers
Themes resolve **project → customer → Metro**. Each layer is a set of design
tokens (CSS custom properties) plus assets. A layer only overrides what it sets.

| Layer | Owned by | Holds |
|---|---|---|
| **Metro base** | Marketing | The portal's own look, built from Metro's two brand efforts (below). Includes type, colour, spacing, surfaces and components. |
| **Customer brand kit** (chain or developer) | Marketing, per customer | Logo variants (light/dark, SVG preferred), primary and secondary colours, optional licensed display font (woff2), banner imagery with focal point, optional pattern or texture |
| **Project theme** | Project rep | Hero and banner images (e.g. property renderings), accent override, and which customer kit applies (e.g. a Marriott property by a developer) |

### Metro's two brand efforts map to different parts of the UI
- **Box & book** (Metro's branded sample box and book) sets the look of the
  product-facing surfaces: the micro-catalog, product detail, sample requests
  and swatches. These are the places where a customer handles the product.
- **Presentation brand** (the PowerPoint structure) sets the page structure:
  title and section-divider layouts, grid, type hierarchy, and how Overview,
  Plan and Quotes are composed. These are the places where Metro presents to
  the customer.

### Rules that keep it usable
- Customer colours **decorate** the hero, headers, accent rules and
  wayfinding. Metro owns the **controls**: buttons, inputs and focus states.
  Every project therefore behaves the same, and a customer's colour can't make
  a button unreadable.
- **Status colours are fixed** (approved, in review, revision) and never taken
  from a brand kit.
- Contrast is checked when a kit is saved. If a brand colour fails against
  text, the portal derives a darker or lighter step for text use and keeps the
  original for decoration.

### Project landing: "Day / Night" split
The project home opens with a hero split on a subtle diagonal (about 8–12°).
- **Day** (left): the customer's side, with their banner image, colours and logo.
- **Night** (right): Metro's side, with the Metro brand and the named rep.

The diagonal carries both logos across the seam. On phones the split stacks,
and the seam becomes a shallow slant between the two bands.

### Brand kit editor (phase 1b)
Marketing uploads logos, colours, fonts and banners per customer, and gets a
live preview of the Day/Night hero in light and dark themes. Projects inherit
from the kit and can override it.

## 3. System context

```mermaid
flowchart LR
  subgraph Users
    C[Brand / Architect / Developer]
    S[Sales & Marketing staff]
  end

  subgraph AWS EC2 (shared initially)
    P[Selections Portal<br/>Laravel · portal.metrofloors.com]
    M[Adobe Commerce<br/>store.metrofloors.com]
    PDB[(portal DB schema)]
    MDB[(magento DB)]
  end

  E[Entra ID + Duo]
  SF[Salesforce]
  M3[Infor M3 13.4]
  S3[(S3: project files)]
  SES[SES email]

  C -- email + password --> P
  S -- OIDC --> E --> P
  P -- customer token / REST --> M
  P -- catalog GraphQL --> M
  P --- PDB
  M --- MDB
  P -- files --> S3
  P -- notifications --> SES
  P -. phase 2 .-> SF
  M -- existing connector --> M3
  P -. phase 3 .-> M3
```

## 4. Identity & security

### 4.1 Staff
- OIDC authorization code flow + PKCE against Entra ID
  (`socialiteproviders/microsoft-azure`).
- Roles are mapped from Entra groups: `portal-sales`, `portal-marketing`,
  `portal-admin`.
- Staff can only reach projects they are assigned to, or all projects for
  `portal-admin`.

### 4.2 Customers
1. A staff member, or a **customer admin** on that project, invites a person
   by email and gives them a project role.
2. The portal calls the Commerce admin REST API (`POST /V1/customers`) with a
   narrowly scoped integration token. It creates the customer in the
   **Project Portal** customer group and sets the `portal_org_id` attribute.
3. The invitee receives a portal-branded invite link. The password is set via
   Commerce (`resetPassword` GraphQL), so Commerce holds the credential.
4. At login, the portal backend exchanges email + password for a customer
   token (`generateCustomerToken`), reads `/V1/customers/me`, confirms the
   group is Project Portal, links the identity and starts its **own** session.
   The browser never holds the Commerce token.
5. Commerce has no customer MFA, so the portal adds its own second factor for
   customers (email one-time code; TOTP optional).

### 4.3 Trade-off to decide before build
Commerce-as-login meets the brief and gives these users one account for the
portal and, later, the store. **But it is the part hardest to port**, because
Magento password hashes can't move to another provider, so a future migration
would mean a password reset for every customer. The alternative is
**Entra External ID** for customers. We already run Entra, it includes MFA,
and it is independent of Commerce. Thanks to D4, either choice can be swapped
later. The question is which cost we'd rather pay.

### 4.4 Application security
- Every project-scoped route passes through a `ProjectPolicy` (membership +
  role). Project slugs are readable but are **not** a secret.
- Files are stored in a private S3 bucket and served only through short-lived
  signed URLs after a policy check. Uploads are checked for type and size and
  virus-scanned (ClamAV).
- An audit log records every approval, rejection, document download, invite
  and quote acceptance.
- Rate-limited login, CSRF protection, a strict CSP, and secure SameSite
  cookies scoped to `portal.metrofloors.com`.

## 5. Domain model (first cut)

| Entity | Key fields |
|--------|-----------|
| `Organization` | name, type (brand, architect, developer, GC, owner), brand_kit |
| `BrandKit` | owner org (or Metro), logos (light/dark), colours, derived text-safe steps, font files, banners (with focal point), pattern, version |
| `User` | uuid, name, email, organization, is_staff |
| `IdentityLink` | user, provider (`entra`, `commerce`), subject |
| `Project` | slug, name, brand, location, stage, sf_opportunity_id, brand_kit (inherited), theme overrides (hero/banner images, accent), welcome text, internal notes |
| `ProjectMember` | project, user, role (`rep`, `customer_admin`, `reviewer`, `viewer`) |
| `Area` | project, name ("Guest Rooms", "Corridors"), quantity + unit (SY/SF) |
| `Selection` | project, area, sku, product snapshot (JSON), status (`proposed`, `in_review`, `approved`, `revise`, `rejected`, `alternate`), qty, notes |
| `Review` | selection, user, decision, comment |
| `SampleRequest` | project, items (selection + qty), ship-to, needed-by, status, commerce_order_id |
| `Document` | project (nullable = global library), category (drawing, spec, submittal, product data), versions |
| `Quote` | project, number, version, status, lines (sku, area, qty, unit, price), pdf, accepted_by/at |
| `Thread` / `Message` | project, subject, optional anchor (selection, document, plan step or quote), body, attachments, **visibility** (`public`, `internal`) |
| `PlanStage` / `PlanStep` | project, order, title, owner, start/due date, status (`not_started`, `on_track`, `at_risk`, `delayed`), done, **is_internal**, milestone flag |
| `BrandTemplate` | parent (e.g. Marriott), optional sub-brand, default areas + units, plan stages and steps (with internal flags and day offsets from the start date), welcome text, pinned library documents, version |
| `Event` | append-only: project, user, type (visit, tab_view, doc_view, download, decision), subject, duration, ts. Rolled up nightly for Engagement. |
| `NotificationPref` | user, event type, channel (`email_immediate`, `email_digest`, `salesforce`, `off`). Salesforce is offered only to staff with a linked Salesforce user id |
| `Activity` | project, actor, verb, subject, for the project feed and audit |

## 6. Integrations

| System | Direction | Phase | How |
|--------|-----------|-------|-----|
| Adobe Commerce (catalog) | read | 1 | GraphQL `products(filter: {sku})`: attributes, media, spec-sheet attachments. Cached, and snapshotted per D7. |
| Adobe Commerce (customers) | read/write | 1 | REST with an integration token limited to customer resources. |
| Adobe Commerce (samples) | write | 2 | Create a $0 order in a **Project Samples** store view, so samples flow to **M3 through the existing connector**, with no new M3 work. |
| Salesforce | two-way | 2 | Connected App, JWT bearer flow. Project ⇄ Opportunity (or a custom `Project__c`). Messages, approvals and sample requests are logged as Activities. Quotes are pulled if they live there. |
| Infor M3 13.4 | read | 3 | Order and shipment status per project, via the existing connector or ION API. |
| SharePoint | read | 2 | **Company library** (spec sheets, EPDs, brand decks, case studies) read from a SharePoint document library via Graph; projects pin items. External users never need SharePoint access: the portal serves the files. |
| SharePoint | write | 3 | Optional mirror of each project's uploaded files to a staff-side site. |
| Salesforce notifications | write | 2 | Staff alerts sent as **Salesforce Custom Notifications** (the `customNotificationAction` REST action): they appear in the bell and the Salesforce mobile app, and tapping one opens the linked Opportunity. Decisions, sample requests and quote acceptance are also logged as Activities on that Opportunity. Portal staff are matched to Salesforce users by Entra email. |

## 6a. Brand templates

A template seeds a new project so that every property of a brand starts from
the same brand standards. It sets the areas, plan stages and steps, a welcome
note and the pinned brand-standard documents. Staff pick one when they
register a project, then edit freely. Changing the template later does not
alter projects already created from it.

| Template | Scope | Notes |
|---|---|---|
| **Marriott** | Parent template, with sub-brand variants added as needed (e.g. select-service vs. full-service) | Marriott's portfolio spans many brands with different finish standards. Start with one parent template and split by sub-brand when Marketing has the standards. |
| **Best Western** | Parent template, with variants for the BWH tiers if needed | BWH brands share a lot of their standards, so one template may cover most properties. |
| **Global Hotel Alliance** | Alliance template, with light defaults | GHA is an alliance of independent brands, so it has fewer common standards. Expect more project-level editing. |
| Blank | – | For independents and one-offs. |

Template contents come from Marketing's brand-standards material and are
**not** invented by the portal. The prototype's template contents are
placeholders.

## 7. Hosting (initial)

- Same EC2 instance as Commerce. Separate nginx vhost `portal.metrofloors.com`,
  **separate PHP-FPM pool and Unix user**, and a separate MySQL schema and DB
  user with **no grants on the Magento schema**.
- Redis uses its own DB index and key prefix. The Laravel queue worker runs
  under Supervisor.
- S3 for files, SES for mail, CloudWatch for logs.
- There is no code or database coupling to Commerce, so moving the portal to
  its own instance or container later is only a deployment change.
- Watch for PHP version drift: Magento 2.4.7 supports PHP 8.2 and 8.3, so pin
  the portal to the same minor version while the two share the box.

## 8. Delivery phases

**Phase 0 — Plan + clickable prototype** *(this branch)*
- This document, plus `prototype/index.html`, a static, clickable walkthrough
  with mock data and a "viewing as" role switcher.

**Phase 1 — MVP (working first version)**
- Laravel skeleton, CI, and deploy to the shared EC2 host
- Staff Entra login; customer Commerce login + second factor; invites
- Projects (staff-created), areas, members, per-project theming, clean URLs
- Server-side viewer filtering + **Client preview** toggle for staff
- Overview: rep welcome, stakeholders, decision tracker, next steps
- Micro-catalog: add a product by SKU from Commerce, review and approve,
  propose alternates
- **Shared plan**: stages, steps, owners, dates, status, internal steps
- Document library with versioning; message threads with attachments and
  **internal notes**
- Staff-only **Internal** tab (handoff notes, internal steps and notes)
- Sample requests (email to the sample desk); quotes (staff upload PDF + lines)
- Event capture for engagement (dashboard follows in phase 2)
- Email notifications (immediate or daily digest) for everyone
- Brand templates: Best Western, Marriott, Global Hotel Alliance

**Phase 2 — Connected**
- Salesforce sync (Internal tab fields, activities); samples as Commerce
  orders → M3; quotes from source system
- Engagement dashboard; staff notifications in Salesforce
- Template editor for Marketing (new brands and sub-brands); SharePoint-backed company library

**Phase 3 — Operational**
- M3 order and shipment tracking; SharePoint mirror; reporting for sales
  leadership; staff "My open steps" across projects
- Optional: project AI assistant grounded in the project's selections, specs
  and threads

## 9. Open questions

1. **Project source of truth:** are projects registered in Salesforce today
   (Opportunity, or a custom object)? If so, the portal should create a
   project from the Salesforce record, not the other way round.
2. **Quotes:** where are they produced (Salesforce CPQ, M3, Excel/PDF)? Do
   customers need to accept them in the portal, or only view them?
3. **Samples:** who fulfils them today, and is a $0 Commerce order → M3 an
   acceptable route?
4. **Custom products:** hospitality projects often use custom carpet designs
   and colorways that aren't in the Commerce catalog. The micro-catalog needs
   portal-native "custom items". Confirm.
5. **Customer identity:** Commerce logins (as briefed) or Entra External ID
   (§4.3)? Should portal customers also be able to shop the store with the
   same login?
6. ~~Reference screenshots~~: received (Aligned reference pack).
7. ~~Migration~~: none. The Aligned rooms were a trial only, so every project
   starts fresh.
8. ~~Notifications~~: email for everyone. Staff also get Salesforce
   notifications (§6).
9. ~~First brand templates~~: Best Western, Marriott, Global Hotel Alliance
   (§6a).
10. **Brand standards content:** who in Marketing owns each brand's standard
    (finish schedules, approved product lists)? Do Marriott templates need to
    split by sub-brand from day one?
11. **Salesforce setup:** a Custom Notification Type and an integration user
    for the portal's Connected App. Who administers the org?
