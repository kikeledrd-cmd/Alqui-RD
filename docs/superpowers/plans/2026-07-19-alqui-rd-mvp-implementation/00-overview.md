# Alqui-RD MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir una PWA inmobiliaria dominicana donde agentes aprobados publiquen propiedades, visitantes las encuentren y contacten, y Alqui-RD registre y gestione la oportunidad hasta visita y cierre.

**Architecture:** Modular monolith with Next.js App Router. Public portal, professional console, and admin console share domain services but remain separated by route groups and authorization policies. Supabase provides PostgreSQL, Auth, and private/public object storage; server actions and route handlers are the only write boundary, and every sensitive action is protected by role checks plus database RLS.

**Tech Stack:** Node.js 22, pnpm 10, Next.js 15, React 19, TypeScript 5, Tailwind CSS 4, Supabase PostgreSQL/Auth/Storage, Zod, React Hook Form, Vitest, Testing Library, Playwright, MapLibre GL, Resend-compatible email adapter, Vercel-compatible cron routes.

## Global Constraints

- Launch territory: Distrito Nacional, Santo Domingo Este, Santo Domingo Norte, and Santo Domingo Oeste.
- Operations: `rent` is primary; `sale` is secondary.
- Property types: apartment, house, room, studio, penthouse, villa.
- Visitors can search, view listings, contact agents, and request visits without registration.
- Registration is required for favorites, saved searches, and persistent comparison lists.
- Agents can register openly but cannot publish until approved and assigned an active plan.
- Public and private addresses must be stored separately; private addresses never leave authorized server responses.
- Location visibility values: `exact`, `approximate`, `sector_only`; default is `approximate`.
- Rental availability confirmation interval: 15 days. Sale confirmation interval: 30 days.
- Manual bank-transfer payments only in the first release; automatic card processing remains behind a provider interface and disabled.
- Mobile-first PWA; no native app in this delivery.
- Out of scope: AI pricing/fraud, digital signatures, internal chat, bank integration, 3D tours, legal contract management, advanced commission engine, nationwide rollout.
- All administrative state changes, role changes, payment approvals, property moderation, and opportunity assignments must create an `audit_logs` record.
- Use test-driven development, one independently testable deliverable per task, and a commit after each task.

---

## Planned File Structure

```text
.
├── .env.example
├── package.json
├── playwright.config.ts
├── vitest.config.ts
├── public/
│   ├── icons/
│   ├── manifest.webmanifest
│   └── offline.html
├── src/
│   ├── app/
│   │   ├── (public)/
│   │   │   ├── page.tsx
│   │   │   ├── buscar/page.tsx
│   │   │   ├── inmueble/[code]/page.tsx
│   │   │   ├── agente/[slug]/page.tsx
│   │   │   ├── tengo-una-propiedad/page.tsx
│   │   │   └── auth/{login,registro,callback}/...
│   │   ├── (professional)/panel/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── propiedades/...
│   │   │   ├── contactos/...
│   │   │   ├── visitas/...
│   │   │   ├── equipo/...
│   │   │   ├── perfil/...
│   │   │   └── plan/...
│   │   ├── (admin)/admin/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── agentes/...
│   │   │   ├── propiedades/...
│   │   │   ├── pagos/...
│   │   │   ├── propietarios/...
│   │   │   └── territorios/...
│   │   ├── api/
│   │   │   ├── leads/whatsapp/route.ts
│   │   │   ├── media/property/route.ts
│   │   │   ├── cron/property-expiration/route.ts
│   │   │   └── health/route.ts
│   │   ├── error.tsx
│   │   ├── layout.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── ui/...
│   │   ├── property/...
│   │   ├── search/...
│   │   └── dashboard/...
│   ├── features/
│   │   ├── auth/
│   │   ├── locations/
│   │   ├── properties/
│   │   ├── media/
│   │   ├── moderation/
│   │   ├── leads/
│   │   ├── visits/
│   │   ├── renewals/
│   │   ├── billing/
│   │   ├── agencies/
│   │   ├── owner-submissions/
│   │   ├── verifications/
│   │   ├── analytics/
│   │   └── audit/
│   ├── lib/
│   │   ├── env.ts
│   │   ├── supabase/{browser,server,admin,middleware}.ts
│   │   ├── auth/authorization.ts
│   │   ├── result.ts
│   │   └── utils.ts
│   ├── middleware.ts
│   └── types/database.ts
├── supabase/
│   ├── config.toml
│   ├── migrations/0001_extensions.sql
│   ├── migrations/0002_identity_and_locations.sql
│   ├── migrations/0003_properties.sql
│   ├── migrations/0004_audit_core_rls.sql
│   ├── migrations/0005_agent_verification_billing.sql
│   ├── migrations/0006_leads_visits.sql
│   ├── migrations/0007_payments_promotions.sql
│   ├── migrations/0008_agencies.sql
│   ├── migrations/0009_owner_submissions.sql
│   ├── migrations/0010_verification_analytics.sql
│   └── seed.sql
└── tests/
    ├── unit/...
    ├── integration/...
    └── e2e/...
```

## Shared Domain Interfaces

Create these contracts early and keep every later task consistent with them:

```ts
export type AppRole =
  | "user"
  | "agent_pending"
  | "agent"
  | "agency_supervisor"
  | "agency_admin"
  | "admin";

export type PropertyOperation = "rent" | "sale";
export type PropertyType =
  | "apartment"
  | "house"
  | "room"
  | "studio"
  | "penthouse"
  | "villa";

export type PropertyStatus =
  | "draft"
  | "in_review"
  | "changes_requested"
  | "published"
  | "confirmation_due"
  | "paused"
  | "reserved"
  | "rented"
  | "sold"
  | "rejected"
  | "archived";

export type LocationVisibility = "exact" | "approximate" | "sector_only";

export type LeadStatus =
  | "new"
  | "contacted"
  | "visit_requested"
  | "visit_scheduled"
  | "visit_completed"
  | "negotiation"
  | "closed"
  | "not_interested"
  | "unreachable"
  | "discarded";

export type ActionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: string; message: string; fields?: Record<string, string[]> } };
```

---
