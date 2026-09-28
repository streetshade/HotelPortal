# Project Selections Portal — Architecture & Delivery Plan

Status: **Draft v0.1** (2026-09-28) · Owner: Digital / Project Sales

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
| `Organization` | name, type (brand, architect, developer, GC, owner) |
| `User` | uuid, name, email, organization, is_staff |
| `IdentityLink` | user, provider (`entra`, `commerce`), subject |
| `Project` | slug, name, brand, location, stage, sf_opportunity_id, theme (logo, accent, hero) |
| `ProjectMember` | project, user, role (`rep`, `customer_admin`, `reviewer`, `viewer`) |
| `Area` | project, name ("Guest Rooms", "Corridors"), quantity + unit (SY/SF) |
| `Selection` | project, area, sku, product snapshot (JSON), status (`proposed`, `in_review`, `approved`, `revise`, `rejected`, `alternate`), qty, notes |
| `Review` | selection, user, decision, comment |
| `SampleRequest` | project, items (selection + qty), ship-to, needed-by, status, commerce_order_id |
| `Document` | project (nullable = global library), category (drawing, spec, submittal, product data), versions |
| `Quote` | project, number, version, status, lines (sku, area, qty, unit, price), pdf, accepted_by/at |
| `Thread` / `Message` | project, subject, optional anchor (selection or document), body, attachments |
| `Activity` | project, actor, verb, subject, for the project feed and audit |

## 6. Integrations

| System | Direction | Phase | How |
|--------|-----------|-------|-----|
| Adobe Commerce (catalog) | read | 1 | GraphQL `products(filter: {sku})`: attributes, media, spec-sheet attachments. Cached, and snapshotted per D7. |
| Adobe Commerce (customers) | read/write | 1 | REST with an integration token limited to customer resources. |
| Adobe Commerce (samples) | write | 2 | Create a $0 order in a **Project Samples** store view, so samples flow to **M3 through the existing connector**, with no new M3 work. |
| Salesforce | two-way | 2 | Connected App, JWT bearer flow. Project ⇄ Opportunity (or a custom `Project__c`). Messages, approvals and sample requests are logged as Activities. Quotes are pulled if they live there. |
| Infor M3 13.4 | read | 3 | Order and shipment status per project, via the existing connector or ION API. |
| SharePoint | optional | 3 | Mirror each project's files to a staff-side SharePoint site via Graph. External users never need SharePoint access. |

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
- Micro-catalog: add a product by SKU from Commerce, review and approve,
  propose alternates
- Document library with versioning; message threads with attachments
- Sample requests (email to the sample desk); quotes (staff upload PDF + lines)
- Email notifications and daily digest

**Phase 2 — Connected**
- Salesforce sync; samples as Commerce orders → M3; quotes from source system

**Phase 3 — Operational**
- M3 order and shipment tracking; SharePoint mirror; reporting for sales
  leadership

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
6. **Reference screenshots** of the TeamAligned portal: project home,
   selections, product view and messages.
