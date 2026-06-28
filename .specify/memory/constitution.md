<!--
  SYNC IMPACT REPORT
  ==================
  Version change: 1.0.0 → 1.1.0
  Bump type: MINOR — new principle added (VII. UI/UX & Design System).

  Principles modified:
  - None. All existing principles (I–VI) are unchanged.

  Principles added:
  - VII. UI/UX & Design System

  Sections added:
  - None.

  Sections removed:
  - None.

  Templates requiring updates:
  - ✅ .specify/templates/plan-template.md — Constitution Check gate references
    this file; no structural changes needed; gate placeholder remains valid.
  - ✅ .specify/templates/spec-template.md — No constitution-specific references;
    template remains valid as-is.
  - ✅ .specify/templates/tasks-template.md — No constitution-specific references;
    template remains valid as-is.

  Deferred TODOs:
  - None. All placeholders resolved.

  --- HISTORICAL RECORD ---
  Version change: [unversioned template] → 1.0.0
  Bump type: MAJOR — first ratification replacing all placeholder content with
             concrete, project-specific principles.
  Principles added (all new): I. Type-Safe Architecture, II. Payment Security &
  Trust Boundary, III. Strict Order Lifecycle, IV. Test Coverage Requirements,
  V. User Experience & Accessibility, VI. Infrastructure & Extensibility.
-->

# Obok Constitution

## Core Principles

### I. Type-Safe Architecture

Next.js App Router with TypeScript strict mode is the required runtime and MUST
be maintained across all application code. The `any` type is PROHIBITED; use
`unknown` with narrowing or define explicit types instead.

Mutations MUST be implemented as Server Actions. API routes are reserved
exclusively for webhooks and third-party callbacks (e.g., Stripe, delivery
aggregators). Mixing mutation logic into API routes violates this boundary.

Prisma is the sole ORM for all database access. Raw SQL is only permitted when
Prisma cannot express the required query (e.g., recursive CTEs, advanced window
functions) and MUST be accompanied by an inline comment explaining why Prisma
was insufficient.

All monetary values MUST be stored as integer cents in every database column,
variable, and computation. Float arithmetic on money values is PROHIBITED.
Currency formatting (e.g., `$12.99`) is performed exclusively at the
presentation/display layer.

`order_items` records MUST snapshot `price_cents` and `item_name` from the
live menu at order-creation time. Price and name changes to menu items MUST
NOT retroactively alter completed or in-progress order records.

### II. Payment Security & Trust Boundary

Every Stripe webhook handler MUST verify the `Stripe-Signature` header using
`stripe.webhooks.constructEvent()` before reading or acting on any payload.
Handlers that skip verification MUST NOT be merged.

Order status transitions MUST only be triggered by verified Stripe webhook
events. Frontend redirects (e.g., `?success=true` query params after Stripe
Checkout) MUST NOT alter order status directly. The webhook is the single
source of truth for payment confirmation.

Admin routes MUST be protected server-side via middleware or Server Component
session checks. Client-only auth guards (e.g., `useEffect` redirects) are
PROHIBITED as a sole protection mechanism for any admin surface.

Environment secrets — API keys, database URLs, Stripe webhook signing secrets,
AWS credentials — MUST NEVER be committed to the repository. AWS Amplify
environment variables are the required mechanism for all credentials in
production and staging environments.

All API boundaries — Server Actions, API route handlers, and webhook payload
processors — MUST validate input with Zod schemas before any business logic
executes. Unvalidated external input reaching business logic is a violation.

### III. Strict Order Lifecycle

The canonical order status transition chain is:

```
pending_payment → paid → accepted → ready → completed
```

Terminal diverging states are `cancelled` and `refunded`. No other statuses
or transition paths are permitted.

Any code that attempts an invalid status transition (e.g., `pending_payment →
accepted`, `completed → paid`) MUST throw a typed error with a descriptive
message identifying the invalid from/to pair. Silent no-ops or status skipping
are PROHIBITED.

Cancellation and refund operations MUST update both Stripe and the local
database atomically. If the operation fails mid-flight, the system MUST
enqueue a retry (e.g., via a background job or idempotent webhook re-delivery)
to prevent permanent drift between Stripe's state and the local DB.

Menu items MUST be soft-deleted by setting `is_active = false`. Hard-deleting
menu item records is PROHIBITED because `order_items` rows reference them to
preserve historical order accuracy.

### IV. Test Coverage Requirements

All money calculation utilities (e.g., subtotal, tax, tip, refund computation)
and all order status transition logic MUST have unit tests. Unit tests MUST
cover normal paths, boundary values, and invalid input (expected throws).

Stripe webhook handling and the complete order creation flow MUST be covered
by integration tests. Integration tests MUST run against a real database
instance (e.g., a local PostgreSQL container or test RDS instance). Mocking
the database in integration tests is PROHIBITED.

Playwright E2E tests MUST cover the following journeys end-to-end:

- **Customer flow**: menu browse → add items to cart → checkout → order
  confirmation page displayed.
- **Admin flow**: new order received → admin accepts order → marks ready →
  marks completed.

Tests MUST be organized so each layer (unit, integration, E2E) can be run
independently without depending on another layer's fixtures or state.

### V. User Experience & Accessibility

Cart operations (add item, remove item, update quantity) MUST use optimistic UI
updates to provide immediate visual feedback before the server response
arrives. The UI MUST reconcile with the server response and roll back on error.

The admin order dashboard MUST surface newly placed orders without requiring a
manual page refresh. Either client-side polling (acceptable at a low interval)
or a real-time subscription (preferred) MUST be implemented.

Menu images MUST be stored in S3 and rendered via the Next.js `<Image>`
component to enable automatic format negotiation (WebP/AVIF) and responsive
sizing. Direct `<img>` tags for menu imagery are PROHIBITED.

All pages and flows MUST be designed mobile-first. The complete checkout flow
(cart review, customer details, payment) MUST be fully operable on a 375 px
viewport without horizontal scrolling or inaccessible tap targets.

Interactive elements MUST use semantic HTML (`<button>`, `<a>`, `<input>`,
etc.). ARIA labels are REQUIRED on interactive elements whose purpose is not
conveyed by visible text alone. The cart drawer/page and the entire checkout
flow MUST be keyboard-navigable without requiring a pointing device.

### VI. Infrastructure & Extensibility

The deployment stack is fixed and MUST NOT be substituted without an explicit
architectural decision record:

- **Hosting**: AWS Amplify
- **Database**: RDS PostgreSQL (accessed exclusively via Prisma)
- **Media storage**: Amazon S3

Transactional emails — order confirmation and cancellation notifications —
MUST be sent via Resend. No other email provider is permitted.

AWS Budgets alerts MUST be configured at **$5**, **$15**, and **$30** monthly
thresholds to detect cost anomalies before they escalate. Deployments to
production without these alerts active are PROHIBITED.

The feature flags `accepting_orders` (boolean) and `estimated_prep_minutes`
(integer) MUST be persisted in a `settings` database table and read at runtime.
Hardcoding these values in source code is PROHIBITED; they must be operator-
configurable without a deployment.

The `orders` table schema MUST accommodate future third-party delivery
aggregator integrations (e.g., DoorDash, Uber Eats) without a destructive
migration. At minimum, nullable `source` (enum or string) and
`aggregator_order_id` fields MUST be present or planned from the initial
migration. New aggregator integrations MUST be addable by extending these
fields, not by restructuring the core orders schema.

### VII. UI/UX & Design System

The frontend component stack is **Next.js + Tailwind CSS + shadcn/ui**. No
alternative component libraries or CSS frameworks may be introduced without an
explicit architectural decision. All new UI work MUST use shadcn/ui primitives
as the baseline before writing custom components.

The design direction is **clean minimalism with a warm, approachable feel**
appropriate for a family restaurant. The color palette and primary/secondary
typography pair MUST be decided and documented before the first feature UI
is built, then applied consistently across every page and component. Ad-hoc
per-component color or font decisions that deviate from the established system
are PROHIBITED without a design-system amendment.

Dark mode is **out of scope for MVP**. No dark mode variants or conditional
dark-mode styling MUST be added until explicitly scheduled.

All interactive states MUST be explicitly designed before a component is
considered complete:

- **hover** — visual affordance change (e.g., color shift, shadow lift)
- **focus** — visible keyboard focus ring meeting WCAG AA
- **loading** — skeleton or spinner; never a frozen UI
- **error** — inline error message adjacent to the failing element
- **empty** — meaningful empty-state copy and/or illustration

Leaving any of these states as the browser default is a UI violation.

WCAG AA contrast ratios are REQUIRED for all text/background pairings.
Semantic HTML and keyboard navigation requirements from Principle V apply here
as well; the design system is the implementation vehicle for those rules.

The `ui-ux-pro-max` skill MUST be invoked for all frontend component design,
implementation, and review work. A frontend pull request MUST NOT be merged
until a `ui-ux-pro-max` review has been completed and any blocking findings
resolved.

## Governance

This constitution is the authoritative governance document for the Obok
restaurant ordering platform. It supersedes all other development practices,
conventions, and ad-hoc decisions. Where a practice conflicts with these
principles, the constitution governs.

**Amendment procedure**: A constitution amendment requires (1) a written
rationale documenting the specific problem the change solves, (2) a version
bump per the semantic versioning rules below, and (3) at least one peer review
before the amendment is merged. Amendments are recorded in the Sync Impact
Report block at the top of this file.

**Versioning policy**:
- MAJOR bump — a principle is removed, redefined in a backward-incompatible
  way, or a breaking governance change is introduced.
- MINOR bump — a new principle or materially new guidance is added.
- PATCH bump — clarifications, wording improvements, or typo fixes that do not
  change the intent of any principle.

**Compliance review**: At the start of every feature's planning phase
(`/speckit-plan`), the Constitution Check gate in `plan.md` MUST be passed
before Phase 1 design begins. The gate MUST be re-verified after Phase 1
design is complete. Any complexity or deviation justified in the Complexity
Tracking table requires explicit sign-off before implementation.

**Version**: 1.1.0 | **Ratified**: 2026-06-27 | **Last Amended**: 2026-06-27
