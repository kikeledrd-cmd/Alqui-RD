# Alqui-RD Prelaunch Landing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir y desplegar una landing profesional de prelanzamiento que explique Alqui-RD, capte tres públicos, permita continuar por WhatsApp y mida conversiones sin presentar el portal inmobiliario como si ya estuviera operativo.

**Architecture:** La landing vivirá en el mismo repositorio que el futuro sistema, pero aislada dentro del dominio `prelaunch`. Next.js renderizará el contenido público y las rutas SEO; los formularios usarán validación compartida, acciones de servidor y un repositorio Supabase protegido. Las interacciones del cliente se limitarán al menú, formularios, pestañas, desplazamiento y analítica.

**Tech Stack:** Node.js 22, pnpm 10, Next.js 15, React 19, TypeScript 5, Tailwind CSS 4, Supabase PostgreSQL, Zod, React Hook Form, Vitest, Testing Library y Playwright.

## Global Constraints

- El idioma visible será español dominicano.
- La web debe mostrar explícitamente: `Plataforma inmobiliaria dominicana en etapa de prelanzamiento.`
- La promesa principal será: `Propiedades reales. Agentes confiables. Todo en un solo lugar.`
- Los CTA principales serán: `Únete a Alqui-RD`, `Soy agente o agencia` y `Tengo una propiedad`.
- No afirmar que la búsqueda, publicación, verificación o gestión de propiedades ya está activa.
- Diseño mobile-first sin desplazamiento horizontal a 320 px.
- Paleta: azul `#12304A`, coral `#D7685D`, arena `#F7F2E8`, blanco `#FFFFFF`, texto secundario `#4F5C63`.
- No solicitar cédula, documentos ni credenciales profesionales en el prelanzamiento.
- No mostrar testimonios, aliados ni métricas que no provengan de datos reales y autorizados.
- Toda entrada se valida en servidor con Zod.
- Las claves administrativas de Supabase solo pueden utilizarse en código marcado como servidor.
- Las tablas de captación no permiten lectura pública.
- La analítica no almacenará nombre, correo, teléfono ni contenido escrito por el usuario.
- Cada tarea termina con pruebas, revisión del diff y un commit independiente.

---

## File Map

```text
src/
  app/
    api/prelaunch/events/route.ts
    blog/[slug]/page.tsx
    blog/page.tsx
    privacidad/page.tsx
    terminos/page.tsx
    globals.css
    layout.tsx
    page.tsx
    robots.ts
    sitemap.ts
  components/
    brand/alqui-rd-logo.tsx
    ui/button.tsx
    ui/container.tsx
    ui/form-field.tsx
  features/prelaunch/
    actions/submit-owner-interest.ts
    actions/submit-professional-application.ts
    actions/submit-property-seeker.ts
    analytics/client.ts
    analytics/events.ts
    analytics/utm.ts
    components/ally-section.tsx
    components/audience-paths.tsx
    components/faq-section.tsx
    components/final-cta.tsx
    components/hero-section.tsx
    components/how-it-works.tsx
    components/owner-interest-form.tsx
    components/problem-section.tsx
    components/professional-application-form.tsx
    components/professional-benefits.tsx
    components/property-seeker-form.tsx
    components/registration-hub.tsx
    components/site-footer.tsx
    components/site-header.tsx
    components/solution-section.tsx
    content.ts
    domain/dedupe.ts
    domain/form-state.ts
    domain/normalize.ts
    domain/schemas.ts
    domain/types.ts
    infrastructure/fingerprint.ts
    infrastructure/prelaunch-repository.ts
    infrastructure/supabase-prelaunch-repository.ts
    infrastructure/whatsapp.ts
  lib/
    env/server.ts
    env/shared.ts
    supabase/admin.ts
    utils/cn.ts
  content/blog/articles.ts
supabase/migrations/202607190001_prelaunch_capture.sql
tests/
  e2e/prelaunch.spec.ts
  unit/prelaunch/*.test.ts
```

---

### Task 1: Scaffold the Application and Quality Tooling

**Files:**
- Create: `package.json`
- Create: `pnpm-lock.yaml`
- Create: `next.config.ts`
- Create: `tsconfig.json`
- Create: `eslint.config.mjs`
- Create: `postcss.config.mjs`
- Create: `vitest.config.ts`
- Create: `vitest.setup.ts`
- Create: `playwright.config.ts`
- Create: `src/app/layout.tsx`
- Create: `src/app/page.tsx`
- Create: `src/app/globals.css`
- Create: `src/lib/utils/cn.ts`
- Create: `.env.example`
- Test: `src/app/page.test.tsx`

**Interfaces:**
- Produces: Next.js App Router project with scripts `dev`, `build`, `start`, `lint`, `typecheck`, `test`, `test:watch`, and `test:e2e`.

- [ ] **Step 1: Create the scaffold without deleting documentation**

Run:

```bash
set -euo pipefail
tmp_dir="$(mktemp -d)"
pnpm dlx create-next-app@15 "$tmp_dir/app" \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --use-pnpm \
  --yes
rsync -a --exclude '.git' "$tmp_dir/app/" ./
rm -rf "$tmp_dir"
```

Expected: existing `docs/` remains and the application files appear at repository root.

- [ ] **Step 2: Install approved dependencies**

Run:

```bash
pnpm add zod@3 react-hook-form@7 @hookform/resolvers@3 @supabase/supabase-js@2 @supabase/ssr@0.6 clsx@2 tailwind-merge@2 server-only@0.0.1
pnpm add -D vitest@3 jsdom@26 @testing-library/react@16 @testing-library/jest-dom@6 @testing-library/user-event@14 @playwright/test@1
```

Expected: lockfile updates successfully.

- [ ] **Step 3: Add test and validation scripts**

Set the `scripts` block in `package.json` to:

```json
{
  "dev": "next dev --turbopack",
  "build": "next build",
  "start": "next start",
  "lint": "next lint",
  "typecheck": "tsc --noEmit",
  "test": "vitest run",
  "test:watch": "vitest",
  "test:e2e": "playwright test"
}
```

If the generated Next.js version no longer supports `next lint`, replace only that script with `eslint .` and document the version-driven change in the task report.

- [ ] **Step 4: Write the failing smoke test**

Create `src/app/page.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import HomePage from "./page";

it("renders the approved prelaunch promise", () => {
  render(<HomePage />);
  expect(
    screen.getByRole("heading", {
      name: "Propiedades reales. Agentes confiables. Todo en un solo lugar.",
    }),
  ).toBeInTheDocument();
});
```

- [ ] **Step 5: Configure Vitest and verify the test fails**

Create `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: {
    environment: "jsdom",
    setupFiles: ["./vitest.setup.ts"],
    globals: true,
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

Create `vitest.setup.ts`:

```ts
import "@testing-library/jest-dom/vitest";
```

Run:

```bash
pnpm test -- src/app/page.test.tsx
```

Expected: FAIL because the generated page does not contain the approved heading.

- [ ] **Step 6: Add the minimal page**

Replace `src/app/page.tsx` with:

```tsx
export default function HomePage() {
  return (
    <main>
      <h1>Propiedades reales. Agentes confiables. Todo en un solo lugar.</h1>
    </main>
  );
}
```

- [ ] **Step 7: Add environment examples**

Create `.env.example`:

```dotenv
NEXT_PUBLIC_SITE_URL=http://localhost:3000
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_ALQUIRD_WHATSAPP_NUMBER=
RATE_LIMIT_SALT=
```

- [ ] **Step 8: Validate and commit**

Run:

```bash
pnpm test -- src/app/page.test.tsx
pnpm typecheck
pnpm lint
pnpm build
git add .
git commit -m "chore: scaffold Alqui-RD prelaunch application"
```

Expected: all commands pass.

---

### Task 2: Create the Brand System and Shared Layout

**Files:**
- Create: `src/components/brand/alqui-rd-logo.tsx`
- Create: `src/components/ui/button.tsx`
- Create: `src/components/ui/container.tsx`
- Create: `src/features/prelaunch/components/site-header.tsx`
- Create: `src/features/prelaunch/components/site-footer.tsx`
- Modify: `src/app/globals.css`
- Modify: `src/app/layout.tsx`
- Test: `src/features/prelaunch/components/site-header.test.tsx`

**Interfaces:**
- Produces: `AlquiRdLogo`, `Button`, `Container`, `SiteHeader`, and `SiteFooter`.

- [ ] **Step 1: Write the failing header test**

```tsx
import { render, screen } from "@testing-library/react";
import { SiteHeader } from "./site-header";

it("exposes the approved navigation and primary CTA", () => {
  render(<SiteHeader />);
  expect(screen.getByRole("link", { name: "Cómo funciona" })).toHaveAttribute("href", "#como-funciona");
  expect(screen.getByRole("link", { name: "Para agentes" })).toHaveAttribute("href", "#para-agentes");
  expect(screen.getByRole("link", { name: "Únete a Alqui-RD" })).toHaveAttribute("href", "#registro");
});
```

Run:

```bash
pnpm test -- src/features/prelaunch/components/site-header.test.tsx
```

Expected: FAIL because `SiteHeader` does not exist.

- [ ] **Step 2: Add theme tokens**

Place in `src/app/globals.css` after `@import "tailwindcss";`:

```css
:root {
  --color-navy: #12304a;
  --color-coral: #d7685d;
  --color-sand: #f7f2e8;
  --color-white: #ffffff;
  --color-muted: #4f5c63;
  --color-border: #d9d2c5;
  --radius-card: 1.5rem;
  --shadow-soft: 0 18px 50px rgb(18 48 74 / 0.08);
}

html { scroll-behavior: smooth; }
body { margin: 0; background: var(--color-sand); color: var(--color-navy); }
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

- [ ] **Step 3: Implement the shared UI**

`src/lib/utils/cn.ts`:

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

`src/components/ui/container.tsx`:

```tsx
import type { PropsWithChildren } from "react";
import { cn } from "@/lib/utils/cn";

export function Container({ children, className }: PropsWithChildren<{ className?: string }>) {
  return <div className={cn("mx-auto w-full max-w-7xl px-5 sm:px-8", className)}>{children}</div>;
}
```

`src/components/ui/button.tsx`:

```tsx
import type { AnchorHTMLAttributes, ButtonHTMLAttributes } from "react";
import { cn } from "@/lib/utils/cn";

const base = "inline-flex min-h-11 items-center justify-center rounded-xl px-5 py-3 font-medium transition focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2";
const variants = {
  primary: "bg-[var(--color-navy)] text-white hover:opacity-90 focus-visible:outline-[var(--color-navy)]",
  coral: "bg-[var(--color-coral)] text-white hover:opacity-90 focus-visible:outline-[var(--color-coral)]",
  outline: "border border-[var(--color-border)] bg-white text-[var(--color-navy)] hover:border-[var(--color-coral)]",
};

type Variant = keyof typeof variants;

export function ButtonLink({ variant = "primary", className, ...props }: AnchorHTMLAttributes<HTMLAnchorElement> & { variant?: Variant }) {
  return <a className={cn(base, variants[variant], className)} {...props} />;
}

export function Button({ variant = "primary", className, ...props }: ButtonHTMLAttributes<HTMLButtonElement> & { variant?: Variant }) {
  return <button className={cn(base, variants[variant], className)} {...props} />;
}
```

- [ ] **Step 4: Implement the brand mark and layout**

Create `AlquiRdLogo` as an accessible inline SVG plus the wordmark `Alqui-RD`; use `aria-label="Alqui-RD"`. Do not approximate the uploaded raster logo as a photographic asset. The SVG must use the approved navy, coral and a small green accent and remain legible at 32 px height.

Create `SiteHeader` with desktop links, an accessible mobile menu button using `aria-expanded`, and the CTA. Create `SiteFooter` with links to `/privacidad`, `/terminos`, `/blog`, contact anchor and the prelanzamiento disclosure.

- [ ] **Step 5: Configure the root layout**

`src/app/layout.tsx` must use `next/font/google`, set `lang="es"`, define default metadata and wrap children with `SiteHeader` and `SiteFooter`.

- [ ] **Step 6: Validate and commit**

Run:

```bash
pnpm test -- src/features/prelaunch/components/site-header.test.tsx
pnpm typecheck
pnpm lint
pnpm build
git add src
 git commit -m "feat: add Alqui-RD brand system and site shell"
```

Expected: all commands pass.

---

### Task 3: Build the Landing Narrative and Approved Sections

**Files:**
- Create: `src/features/prelaunch/content.ts`
- Create: `src/features/prelaunch/components/hero-section.tsx`
- Create: `src/features/prelaunch/components/problem-section.tsx`
- Create: `src/features/prelaunch/components/solution-section.tsx`
- Create: `src/features/prelaunch/components/audience-paths.tsx`
- Create: `src/features/prelaunch/components/how-it-works.tsx`
- Create: `src/features/prelaunch/components/professional-benefits.tsx`
- Create: `src/features/prelaunch/components/ally-section.tsx`
- Create: `src/features/prelaunch/components/faq-section.tsx`
- Create: `src/features/prelaunch/components/final-cta.tsx`
- Modify: `src/app/page.tsx`
- Test: `src/features/prelaunch/components/landing-content.test.tsx`

**Interfaces:**
- Produces: all static sections and `prelaunchContent` as the single source of approved copy.

- [ ] **Step 1: Write the failing content test**

```tsx
import { render, screen } from "@testing-library/react";
import HomePage from "@/app/page";

it("communicates prelanzamiento without claiming the platform is live", () => {
  render(<HomePage />);
  expect(screen.getByText("Plataforma inmobiliaria dominicana en etapa de prelanzamiento.")).toBeInTheDocument();
  expect(screen.getByText("Buscar una propiedad no debería significar navegar entre anuncios vencidos, información incompleta y contactos sin respuesta.")).toBeInTheDocument();
  expect(screen.queryByText(/miles de propiedades disponibles/i)).not.toBeInTheDocument();
});
```

- [ ] **Step 2: Create the content source**

`src/features/prelaunch/content.ts` must export immutable objects containing exactly:

```ts
export const prelaunchContent = {
  hero: {
    eyebrow: "Plataforma inmobiliaria dominicana en etapa de prelanzamiento.",
    title: "Propiedades reales. Agentes confiables. Todo en un solo lugar.",
    description: "Alqui-RD conecta personas, propietarios y profesionales inmobiliarios en una plataforma dominicana diseñada para encontrar, publicar y gestionar propiedades con mayor confianza.",
  },
  problem: {
    title: "Buscar una propiedad debería ser más claro y confiable.",
    description: "Buscar una propiedad no debería significar navegar entre anuncios vencidos, información incompleta y contactos sin respuesta.",
    items: ["Propiedades que ya no están disponibles", "Información dispersa entre redes y WhatsApp", "Falta de confianza y seguimiento"],
  },
  solution: {
    title: "Una experiencia diseñada para el mercado dominicano.",
    description: "Alqui-RD reunirá propiedades, agentes y oportunidades en un entorno diseñado para el mercado dominicano.",
    pillars: ["Inventario actualizado", "Profesionales verificados", "Contacto y visitas organizadas"],
  },
  process: ["Busca", "Compara", "Conecta", "Visita", "Encuentra tu próximo espacio"],
} as const;
```

- [ ] **Step 3: Implement semantic sections**

Each section must:

- Use a unique `id` matching the navigation.
- Use one visible `h2`.
- Avoid hard-coded metrics.
- Use server components unless interaction is required.
- Use CSS and simple inline SVG icons rather than a large icon dependency.

The hero CTA links must be:

```tsx
<ButtonLink href="#registro" data-audience="seeker">Únete a Alqui-RD</ButtonLink>
<ButtonLink href="#registro" variant="coral" data-audience="professional">Soy agente o agencia</ButtonLink>
```

The owner CTA must link to `#registro` with `data-audience="owner"`.

- [ ] **Step 4: Add the ally section with truthful empty state**

Before real allies exist, render:

```tsx
<p>Estamos formando la primera red de aliados fundadores de Alqui-RD.</p>
<p>Los perfiles aparecerán aquí únicamente con autorización.</p>
```

Do not render fake logos, names, avatars or counts.

- [ ] **Step 5: Assemble the page and validate**

Run:

```bash
pnpm test -- src/features/prelaunch/components/landing-content.test.tsx
pnpm typecheck
pnpm lint
pnpm build
git add src
 git commit -m "feat: build Alqui-RD prelaunch landing narrative"
```

---

### Task 4: Define Form Domains, Validation and Normalization

**Files:**
- Create: `src/features/prelaunch/domain/types.ts`
- Create: `src/features/prelaunch/domain/schemas.ts`
- Create: `src/features/prelaunch/domain/normalize.ts`
- Create: `src/features/prelaunch/domain/dedupe.ts`
- Create: `src/features/prelaunch/domain/form-state.ts`
- Test: `tests/unit/prelaunch/schemas.test.ts`
- Test: `tests/unit/prelaunch/normalize.test.ts`

**Interfaces:**
- Produces: `propertySeekerSchema`, `professionalApplicationSchema`, `ownerInterestSchema`, `normalizePhone`, `normalizeEmail`, `createDedupeKey`, and `PrelaunchFormState`.

- [ ] **Step 1: Write failing schema tests**

```ts
import { propertySeekerSchema, professionalApplicationSchema } from "@/features/prelaunch/domain/schemas";

it("accepts a valid property seeker", () => {
  expect(propertySeekerSchema.safeParse({
    name: "Ana Pérez",
    whatsapp: "809-555-0101",
    email: "ana@example.com",
    zoneInterest: "Santo Domingo Este",
    operation: "rent",
    consent: true,
    website: "",
    startedAt: Date.now() - 4000,
  }).success).toBe(true);
});

it("requires an agency name for an agency profile", () => {
  const result = professionalApplicationSchema.safeParse({
    name: "Carlos Ruiz",
    whatsapp: "8295550101",
    email: "carlos@example.com",
    profileType: "agency",
    agencyName: "",
    workZones: "Distrito Nacional",
    propertyCountRange: "10-20",
    consent: true,
    website: "",
    startedAt: Date.now() - 4000,
  });
  expect(result.success).toBe(false);
});
```

- [ ] **Step 2: Define exact domain types**

```ts
export type SeekerOperation = "rent" | "buy";
export type ProfessionalProfileType = "independent_agent" | "agency";
export type PropertyCountRange = "1-5" | "6-10" | "10-20" | "21-50" | "50+";
export type OwnerOperation = "rent" | "sell";
export type OwnerPropertyType = "apartment" | "house" | "room" | "studio" | "penthouse" | "villa";

export type PrelaunchFormState = {
  status: "idle" | "success" | "error" | "duplicate" | "rate_limited";
  message: string;
  fieldErrors?: Record<string, string[]>;
};
```

- [ ] **Step 3: Implement Zod schemas**

Rules:

- Name: 2–100 characters.
- WhatsApp: after normalization, 10 digits for Dominican local numbers or 11 digits when prefixed by `1`.
- Email: lowercased valid email.
- Zone fields: 2–120 characters.
- Consent must be literal `true`.
- `website` is the honeypot and must remain empty.
- `startedAt` must represent at least 1500 ms before validation.
- Agency name is required only when `profileType` is `agency`.

- [ ] **Step 4: Implement normalization and dedupe**

```ts
import { createHash } from "node:crypto";

export function normalizeEmail(value: string) {
  return value.trim().toLowerCase();
}

export function normalizePhone(value: string) {
  const digits = value.replace(/\D/g, "");
  return digits.length === 10 ? `1${digits}` : digits;
}

export function createDedupeKey(kind: "seeker" | "professional" | "owner", email: string | undefined, phone: string) {
  return createHash("sha256")
    .update(`${kind}:${normalizeEmail(email ?? "")}:${normalizePhone(phone)}`)
    .digest("hex");
}
```

- [ ] **Step 5: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/schemas.test.ts tests/unit/prelaunch/normalize.test.ts
pnpm typecheck
pnpm lint
git add src/features/prelaunch/domain tests/unit/prelaunch
 git commit -m "feat: define prelaunch capture validation domain"
```

---

### Task 5: Add Supabase Capture Storage and Server-Only Repository

**Files:**
- Create: `supabase/migrations/202607190001_prelaunch_capture.sql`
- Create: `src/lib/env/server.ts`
- Create: `src/lib/env/shared.ts`
- Create: `src/lib/supabase/admin.ts`
- Create: `src/features/prelaunch/infrastructure/prelaunch-repository.ts`
- Create: `src/features/prelaunch/infrastructure/supabase-prelaunch-repository.ts`
- Create: `src/features/prelaunch/infrastructure/fingerprint.ts`
- Test: `tests/unit/prelaunch/repository.test.ts`

**Interfaces:**
- Produces: `PrelaunchRepository` with `createSeeker`, `createProfessional`, `createOwner`, `recordEvent`, and `allowSubmission`.

- [ ] **Step 1: Write the repository contract test with a fake implementation**

```ts
import type { PrelaunchRepository } from "@/features/prelaunch/infrastructure/prelaunch-repository";

it("returns duplicate when the same dedupe key is inserted twice", async () => {
  const seen = new Set<string>();
  const repository: PrelaunchRepository = {
    async createSeeker(input) {
      if (seen.has(input.dedupeKey)) return "duplicate";
      seen.add(input.dedupeKey);
      return "created";
    },
    async createProfessional() { return "created"; },
    async createOwner() { return "created"; },
    async recordEvent() { return undefined; },
    async allowSubmission() { return true; },
  };
  expect(await repository.createSeeker({ dedupeKey: "a" } as never)).toBe("created");
  expect(await repository.createSeeker({ dedupeKey: "a" } as never)).toBe("duplicate");
});
```

- [ ] **Step 2: Create the migration**

The SQL file must create:

- `prelaunch_property_seekers`.
- `prelaunch_professional_applications`.
- `prelaunch_owner_interests`.
- `prelaunch_events`.
- `prelaunch_rate_limits`.
- Function `allow_prelaunch_submission`.

All tables must use UUID primary keys, `created_at timestamptz default now()`, `dedupe_key text unique` for the three capture tables, UTM columns, `consent boolean not null`, and RLS enabled without anon/authenticated read policies.

The rate-limit function must:

- Bucket fingerprints in 15-minute windows.
- Allow at most five accepted attempts per bucket.
- Use `security definer` and `set search_path = public`.
- Revoke execution from `public`, `anon`, and `authenticated`.
- Grant execution only to `service_role`.

- [ ] **Step 3: Validate environment variables**

`src/lib/env/server.ts`:

```ts
import "server-only";
import { z } from "zod";

const schema = z.object({
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(20),
  RATE_LIMIT_SALT: z.string().min(16),
});

export function getServerEnv() {
  return schema.parse(process.env);
}
```

`src/lib/env/shared.ts` validates `NEXT_PUBLIC_SITE_URL` and permits an empty WhatsApp number during local build while returning `null` from a getter.

- [ ] **Step 4: Implement the admin client and repository**

The admin client must import `server-only`, disable session persistence and never be exported from a client component.

The Supabase repository must use `upsert(..., { onConflict: "dedupe_key", ignoreDuplicates: true })` and return `created` or `duplicate` deterministically.

- [ ] **Step 5: Implement anonymous fingerprinting**

Use SHA-256 over normalized IP, user agent, route kind and `RATE_LIMIT_SALT`. Never store the raw IP or user agent in the database.

- [ ] **Step 6: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/repository.test.ts
pnpm typecheck
pnpm lint
git add supabase src/lib src/features/prelaunch/infrastructure tests/unit/prelaunch/repository.test.ts
 git commit -m "feat: add secure prelaunch capture persistence"
```

---

### Task 6: Implement Server Actions and Accessible Capture Forms

**Files:**
- Create: `src/features/prelaunch/actions/submit-property-seeker.ts`
- Create: `src/features/prelaunch/actions/submit-professional-application.ts`
- Create: `src/features/prelaunch/actions/submit-owner-interest.ts`
- Create: `src/components/ui/form-field.tsx`
- Create: `src/features/prelaunch/components/property-seeker-form.tsx`
- Create: `src/features/prelaunch/components/professional-application-form.tsx`
- Create: `src/features/prelaunch/components/owner-interest-form.tsx`
- Create: `src/features/prelaunch/components/registration-hub.tsx`
- Modify: `src/app/page.tsx`
- Test: `tests/unit/prelaunch/actions.test.ts`
- Test: `src/features/prelaunch/components/registration-hub.test.tsx`

**Interfaces:**
- Produces: three server actions returning `PrelaunchFormState` and a registration hub with tabs `seeker`, `professional`, and `owner`.

- [ ] **Step 1: Write failing component behavior tests**

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { RegistrationHub } from "./registration-hub";

it("switches from seeker to professional form", async () => {
  render(<RegistrationHub />);
  expect(screen.getByRole("heading", { name: "Únete a la lista de espera" })).toBeInTheDocument();
  await userEvent.click(screen.getByRole("tab", { name: "Soy agente o agencia" }));
  expect(screen.getByRole("heading", { name: "Conviértete en aliado fundador" })).toBeInTheDocument();
});
```

- [ ] **Step 2: Implement a shared action pipeline**

Each action must:

1. Parse `FormData` into a plain object.
2. Validate through its Zod schema.
3. Reject a non-empty honeypot with a generic success message.
4. Obtain headers and calculate the anonymous fingerprint.
5. Call `allowSubmission` before inserting.
6. Normalize fields and generate the dedupe key.
7. Insert through `PrelaunchRepository`.
8. Return `success`, `duplicate`, `rate_limited`, or `error` without throwing validation details to the browser.

A duplicate response is friendly and does not reveal which field matched:

```ts
{
  status: "duplicate",
  message: "Ya tenemos un registro relacionado con estos datos. Te mantendremos informado.",
}
```

- [ ] **Step 3: Implement forms with React Hook Form**

Use `zodResolver`, visible labels, inline errors, `aria-invalid`, `aria-describedby`, an `aria-live="polite"` status region, disabled pending button and native autocomplete attributes.

The hidden fields must be:

```tsx
<input type="text" tabIndex={-1} autoComplete="off" className="hidden" aria-hidden="true" {...register("website")} />
<input type="hidden" value={startedAt} {...register("startedAt", { valueAsNumber: true })} />
```

- [ ] **Step 4: Implement CTA-driven tab selection**

`RegistrationHub` must read `audience` from the URL hash query fragment generated by CTA clicks, or receive a custom browser event. Use one documented mechanism consistently. A keyboard user activating a CTA must arrive at the matching tab and focus the form heading.

- [ ] **Step 5: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/actions.test.ts src/features/prelaunch/components/registration-hub.test.tsx
pnpm typecheck
pnpm lint
pnpm build
git add src tests
 git commit -m "feat: add prelaunch capture forms and server actions"
```

---

### Task 7: Add Success States and WhatsApp Continuation

**Files:**
- Create: `src/features/prelaunch/infrastructure/whatsapp.ts`
- Create: `src/features/prelaunch/components/registration-success.tsx`
- Modify: the three form components
- Test: `tests/unit/prelaunch/whatsapp.test.ts`
- Test: `src/features/prelaunch/components/registration-success.test.tsx`

**Interfaces:**
- Produces: `buildWhatsAppUrl(audience)` and audience-specific success UI.

- [ ] **Step 1: Write failing URL tests**

```ts
import { buildWhatsAppUrl } from "@/features/prelaunch/infrastructure/whatsapp";

it("builds the professional founder message", () => {
  expect(buildWhatsAppUrl("professional", "18095550101")).toBe(
    "https://wa.me/18095550101?text=Hola%2C%20complet%C3%A9%20la%20solicitud%20para%20ser%20aliado%20fundador%20de%20Alqui-RD.",
  );
});
```

- [ ] **Step 2: Implement exact audience messages**

```ts
const messages = {
  seeker: "Hola, me registré en la lista de espera de Alqui-RD y quiero recibir información sobre propiedades.",
  professional: "Hola, completé la solicitud para ser aliado fundador de Alqui-RD.",
  owner: "Hola, registré mi interés para publicar una propiedad con Alqui-RD.",
} as const;
```

Normalize the configured number to digits. Return `null` when no valid number exists so the UI can hide the WhatsApp action without breaking registration.

- [ ] **Step 3: Implement success states**

Render exact approved messages for seeker, professional and owner. Provide links to the page start and relevant section. The WhatsApp link must use `target="_blank"` and `rel="noreferrer"`.

- [ ] **Step 4: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/whatsapp.test.ts src/features/prelaunch/components/registration-success.test.tsx
pnpm typecheck
pnpm lint
git add src/features/prelaunch
 git commit -m "feat: add WhatsApp continuation after registration"
```

---

### Task 8: Add Blog, Legal Pages and Initial Editorial Content

**Files:**
- Create: `src/content/blog/articles.ts`
- Create: `src/app/blog/page.tsx`
- Create: `src/app/blog/[slug]/page.tsx`
- Create: `src/app/privacidad/page.tsx`
- Create: `src/app/terminos/page.tsx`
- Test: `tests/unit/prelaunch/blog.test.ts`

**Interfaces:**
- Produces: `articles`, `getArticleBySlug`, static article routes, privacy and terms routes.

- [ ] **Step 1: Write failing article registry tests**

```ts
import { articles, getArticleBySlug } from "@/content/blog/articles";

it("ships the three approved launch articles", () => {
  expect(articles.map((article) => article.slug)).toEqual([
    "evitar-anuncios-inmobiliarios-desactualizados",
    "que-revisar-antes-de-alquilar-en-republica-dominicana",
    "como-alqui-rd-ayudara-a-los-agentes",
  ]);
});

it("returns undefined for an unknown slug", () => {
  expect(getArticleBySlug("desconocido")).toBeUndefined();
});
```

- [ ] **Step 2: Define the article type**

```ts
export type BlogSection = { heading: string; paragraphs: string[] };
export type BlogArticle = {
  slug: string;
  title: string;
  description: string;
  publishedAt: string;
  category: "Búsqueda" | "Alquiler" | "Profesionales";
  readingMinutes: number;
  sections: BlogSection[];
};
```

- [ ] **Step 3: Add the approved editorial pieces**

Each article must contain at least four substantive sections and 700 words of original Spanish copy. The copy must avoid legal guarantees, fabricated statistics and claims that Alqui-RD is already operating. Use these exact titles:

1. `Cómo evitar anuncios inmobiliarios desactualizados al buscar vivienda`.
2. `Qué debe revisar una persona antes de alquilar una propiedad en República Dominicana`.
3. `Cómo Alqui-RD ayudará a los agentes a organizar propiedades y oportunidades`.

- [ ] **Step 4: Add static generation and metadata**

`generateStaticParams` must return every slug. Unknown slugs call `notFound()`. Each article defines title, description, canonical URL and Article JSON-LD.

- [ ] **Step 5: Add privacy and terms**

The privacy page must explain captured fields, purposes, consent, UTM/referrer data, retention review, no sale of personal data, contact path and the use of WhatsApp as an external service. The terms page must clarify prelanzamiento status, no brokerage guarantee, no active property listing service, acceptable use and future updates.

- [ ] **Step 6: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/blog.test.ts
pnpm typecheck
pnpm lint
pnpm build
git add src/app/blog src/app/privacidad src/app/terminos src/content
 git commit -m "feat: add Alqui-RD prelaunch blog and legal pages"
```

---

### Task 9: Implement First-Party Conversion Analytics and UTM Capture

**Files:**
- Create: `src/features/prelaunch/analytics/events.ts`
- Create: `src/features/prelaunch/analytics/utm.ts`
- Create: `src/features/prelaunch/analytics/client.ts`
- Create: `src/app/api/prelaunch/events/route.ts`
- Modify: CTA, forms and article links
- Test: `tests/unit/prelaunch/analytics.test.ts`

**Interfaces:**
- Produces: `PrelaunchEventName`, `readUtmContext`, `trackPrelaunchEvent`, and POST `/api/prelaunch/events`.

- [ ] **Step 1: Write failing whitelist tests**

```ts
import { prelaunchEventSchema } from "@/features/prelaunch/analytics/events";

it("rejects arbitrary event names and personal fields", () => {
  expect(prelaunchEventSchema.safeParse({ event: "email_captured", email: "a@b.com" }).success).toBe(false);
});
```

- [ ] **Step 2: Define the event whitelist**

Use only:

```ts
export const prelaunchEventNames = [
  "prelaunch_cta_join_clicked",
  "prelaunch_cta_professional_clicked",
  "prelaunch_cta_owner_clicked",
  "prelaunch_form_started",
  "prelaunch_form_submitted",
  "prelaunch_form_succeeded",
  "prelaunch_form_failed",
  "prelaunch_whatsapp_clicked",
  "prelaunch_blog_article_opened",
] as const;
```

The payload permits `audience`, `source`, `utmSource`, `utmMedium`, `utmCampaign`, `path`, and `articleSlug`; it rejects extra keys using `.strict()`.

- [ ] **Step 3: Implement UTM capture**

Read the current URL once, store only UTM and referrer context in `sessionStorage`, and never store form values. Provide a server-safe fallback returning empty context.

- [ ] **Step 4: Implement the API route**

Validate JSON, apply the anonymous rate limiter, record through `PrelaunchRepository`, return `202` for accepted events and `400` for invalid payloads. Never echo database errors.

- [ ] **Step 5: Instrument conversion points**

Track CTA click, form start on first field interaction, submit attempt, action result, WhatsApp click and article open. Analytics failure must never block a form or navigation.

- [ ] **Step 6: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/analytics.test.ts
pnpm typecheck
pnpm lint
pnpm build
git add src/features/prelaunch/analytics src/app/api src/features/prelaunch/components src/app/blog
 git commit -m "feat: add privacy-safe prelaunch conversion analytics"
```

---

### Task 10: Add SEO, Structured Data and Social Metadata

**Files:**
- Create: `src/app/sitemap.ts`
- Create: `src/app/robots.ts`
- Create: `src/features/prelaunch/components/structured-data.tsx`
- Modify: `src/app/layout.tsx`
- Modify: `src/app/page.tsx`
- Modify: blog routes
- Test: `tests/unit/prelaunch/seo.test.tsx`

**Interfaces:**
- Produces: metadata for every public route and structured data components.

- [ ] **Step 1: Write failing metadata tests**

Test that the homepage title contains `Alqui-RD` and `prelanzamiento`, and that the FAQ JSON-LD includes the six approved questions.

- [ ] **Step 2: Configure canonical metadata**

The root metadata must use:

```ts
{
  title: "Alqui-RD | Plataforma inmobiliaria dominicana en prelanzamiento",
  description: "Únete al prelanzamiento de Alqui-RD, una plataforma dominicana para conectar personas, propietarios, agentes y agencias con mayor confianza.",
}
```

Use `metadataBase` from `NEXT_PUBLIC_SITE_URL`, Open Graph, Twitter Cards and a generated social image or a checked-in optimized asset.

- [ ] **Step 3: Add structured data**

Render `Organization`, `WebSite`, and `FAQPage` on the homepage. Render `Article` on blog detail pages. Do not use `RealEstateAgent` until a real organization profile and operating details are available.

- [ ] **Step 4: Add sitemap and robots**

Include `/`, `/blog`, all article slugs, `/privacidad`, and `/terminos`. Allow indexing of public routes and disallow `/api/`.

- [ ] **Step 5: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/seo.test.tsx
pnpm typecheck
pnpm lint
pnpm build
git add src/app src/features/prelaunch/components/structured-data.tsx
 git commit -m "feat: add SEO and structured data for prelaunch"
```

---

### Task 11: Harden Accessibility, Security and Performance

**Files:**
- Modify: `next.config.ts`
- Modify: interactive components and styles
- Create: `tests/unit/prelaunch/security-headers.test.ts`
- Create: `tests/unit/prelaunch/accessibility.test.tsx`

**Interfaces:**
- Produces: security headers and accessible, reduced-motion-compatible UI.

- [ ] **Step 1: Add security headers**

Configure at least:

```ts
const securityHeaders = [
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
];
```

Add a Content Security Policy compatible with Next.js and Supabase. Validate it in production mode; do not use `unsafe-eval` in production.

- [ ] **Step 2: Audit keyboard and focus behavior**

Verify:

- Mobile menu traps no focus and closes with Escape.
- Tabs use `role="tablist"`, `role="tab"`, `aria-selected`, `aria-controls`, and arrow-key navigation.
- CTA-driven scrolling focuses the selected form heading.
- Error summary links focus invalid fields.
- Success messages use `role="status"`.

- [ ] **Step 3: Optimize media and client boundaries**

Use `next/image` with explicit sizes. Keep static sections as server components. Import no map or dashboard library. Ensure no hero image exceeds 300 KB after optimization.

- [ ] **Step 4: Run production audits**

Run the production app and Lighthouse against the homepage and one article. Record scores in `docs/quality/prelaunch-lighthouse.md`. Required minimum: 90 for Performance, Accessibility, Best Practices and SEO in a production build.

- [ ] **Step 5: Validate and commit**

Run:

```bash
pnpm test -- tests/unit/prelaunch/security-headers.test.ts tests/unit/prelaunch/accessibility.test.tsx
pnpm typecheck
pnpm lint
pnpm build
git add next.config.ts src tests docs/quality
 git commit -m "fix: harden prelaunch accessibility security and performance"
```

---

### Task 12: Add End-to-End Flows and Launch Documentation

**Files:**
- Create: `tests/e2e/prelaunch.spec.ts`
- Create: `docs/operations/prelaunch-deployment.md`
- Create: `docs/operations/prelaunch-content-operations.md`
- Create: `docs/operations/prelaunch-launch-checklist.md`
- Modify: `README.md`

**Interfaces:**
- Produces: verified user journeys and deployment instructions.

- [ ] **Step 1: Configure Playwright**

Use Chromium, start `pnpm dev`, base URL `http://127.0.0.1:3000`, traces on first retry and screenshots only on failure.

- [ ] **Step 2: Write complete E2E journeys**

Cover:

1. Homepage shows approved promise and prelanzamiento notice.
2. `Únete a Alqui-RD` selects seeker form and focuses its heading.
3. Client validation rejects incomplete seeker submission.
4. A mocked successful seeker action renders the seeker success message.
5. Professional CTA selects the professional form.
6. Agency selection requires agency name.
7. Owner CTA selects owner form.
8. Mobile menu opens, navigates and closes.
9. Blog index opens all three articles.
10. Privacy and terms routes load.
11. Unknown blog slug returns 404.
12. No page has horizontal overflow at 320 px.

For tests that would write to Supabase, intercept the action or inject a test repository; do not depend on production credentials.

- [ ] **Step 3: Write deployment documentation**

`prelaunch-deployment.md` must include:

- Node 22 and pnpm 10 requirements.
- Supabase project setup.
- Migration command.
- Environment variables and their public/private classification.
- Vercel deployment steps.
- Custom domain and HTTPS verification.
- Rollback steps.

`prelaunch-content-operations.md` must explain how to add an article, publish a real ally with written consent, update FAQs and enable real metrics.

`prelaunch-launch-checklist.md` must include copy review, WhatsApp test, consent review, database verification, analytics verification, Lighthouse results, mobile testing, social image, sitemap, robots and backup/rollback.

- [ ] **Step 4: Run the complete release gate**

Run:

```bash
pnpm test
pnpm typecheck
pnpm lint
pnpm build
pnpm test:e2e
```

Expected: all commands pass with zero skipped critical journeys.

- [ ] **Step 5: Review scope and commit**

Confirm the final diff contains no login, property search, operational listing, payment, agent dashboard or fabricated metric. Then run:

```bash
git add tests/e2e docs/operations README.md
 git commit -m "test: verify prelaunch landing release journeys"
```

---

## Final Verification

Before opening the pull request:

```bash
pnpm install --frozen-lockfile
pnpm test
pnpm typecheck
pnpm lint
pnpm build
pnpm test:e2e
git status --short
```

Expected:

- All checks pass.
- `git status --short` is empty.
- The homepage clearly says prelanzamiento.
- Three capture flows persist securely.
- No personal data appears in analytics events.
- WhatsApp messages match the approved audience.
- Three blog articles, privacy, terms, sitemap and robots are available.
- No functionality from the operational marketplace was implemented prematurely.
