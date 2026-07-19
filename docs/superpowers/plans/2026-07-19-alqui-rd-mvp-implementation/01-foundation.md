# Ola 1 — Foundation

### Task 1: Project Foundation and Quality Gates

**Files:**
- Create: `package.json`
- Create: `.env.example`
- Create: `vitest.config.ts`
- Create: `playwright.config.ts`
- Create: `src/lib/env.ts`
- Create: `src/lib/result.ts`
- Create: `src/app/api/health/route.ts`
- Create: `tests/unit/health.test.ts`

**Interfaces:**
- Consumes: none.
- Produces: `env`, `ActionResult<T>`, test scripts, and `/api/health` returning `{ status: "ok" }`.

- [ ] **Step 1: Scaffold the Next.js application in the existing repository**

Run:

```bash
cd /mnt/data/Alqui-RD
rm -rf /tmp/alqui-rd-web
pnpm dlx create-next-app@15 /tmp/alqui-rd-web --typescript --eslint --tailwind --app --src-dir --import-alias '@/*' --use-pnpm
rsync -a --exclude='.git' /tmp/alqui-rd-web/ ./
rm -rf /tmp/alqui-rd-web
pnpm add @supabase/ssr @supabase/supabase-js zod react-hook-form @hookform/resolvers clsx tailwind-merge lucide-react maplibre-gl
pnpm add -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @playwright/test
```

Expected: Next.js project files exist and `pnpm build` succeeds.

- [ ] **Step 2: Add deterministic scripts and environment contract**

Set these scripts in `package.json`:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "verify": "pnpm lint && pnpm typecheck && pnpm test && pnpm build"
  }
}
```

Create `.env.example`:

```dotenv
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
CRON_SECRET=
EMAIL_FROM=Alqui-RD <notificaciones@alqui-rd.com>
```

- [ ] **Step 3: Write the failing health endpoint test**

Create `tests/unit/health.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { GET } from "@/app/api/health/route";

describe("GET /api/health", () => {
  it("returns an ok payload", async () => {
    const response = await GET();
    expect(response.status).toBe(200);
    await expect(response.json()).resolves.toEqual({ status: "ok" });
  });
});
```

- [ ] **Step 4: Run the test and verify failure**

Run:

```bash
pnpm test tests/unit/health.test.ts
```

Expected: FAIL because `src/app/api/health/route.ts` does not exist.

- [ ] **Step 5: Implement the health endpoint and shared result type**

Create `src/app/api/health/route.ts`:

```ts
import { NextResponse } from "next/server";

export function GET() {
  return NextResponse.json({ status: "ok" as const });
}
```

Create `src/lib/result.ts`:

```ts
export type ActionResult<T> =
  | { ok: true; data: T }
  | {
      ok: false;
      error: {
        code: string;
        message: string;
        fields?: Record<string, string[]>;
      };
    };
```

Create `src/lib/env.ts`:

```ts
import { z } from "zod";

const schema = z.object({
  NEXT_PUBLIC_APP_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
  CRON_SECRET: z.string().min(24),
  EMAIL_FROM: z.string().min(3),
});

export const env = schema.parse(process.env);
```

- [ ] **Step 6: Run quality gates**

Run:

```bash
pnpm test tests/unit/health.test.ts
pnpm typecheck
pnpm build
```

Expected: all commands exit with status 0.

- [ ] **Step 7: Commit**

```bash
git add package.json pnpm-lock.yaml .env.example vitest.config.ts playwright.config.ts src tests
git commit -m "chore: scaffold Alqui-RD application"
```

---

### Task 2: Supabase Schema, Enums, and Dominican Territory Seed

**Files:**
- Create: `supabase/config.toml`
- Create: `supabase/migrations/0001_extensions.sql`
- Create: `supabase/migrations/0002_identity_and_locations.sql`
- Create: `supabase/migrations/0003_properties.sql`
- Create: `supabase/seed.sql`
- Create: `src/types/database.ts`
- Create: `tests/integration/schema.test.ts`

**Interfaces:**
- Consumes: Supabase CLI and shared enum names.
- Produces: tables `profiles`, `agent_profiles`, `locations`, `properties`, `property_media`, `property_features`; generated `Database` type.

- [ ] **Step 1: Initialize local Supabase and add the schema smoke test**

Run:

```bash
pnpm add -D supabase
pnpm supabase init
```

Create `tests/integration/schema.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { createClient } from "@supabase/supabase-js";

const url = process.env.NEXT_PUBLIC_SUPABASE_URL!;
const key = process.env.SUPABASE_SERVICE_ROLE_KEY!;

const db = createClient(url, key, { auth: { persistSession: false } });

describe("database schema", () => {
  it("contains the initial locations", async () => {
    const { data, error } = await db
      .from("locations")
      .select("name,kind")
      .in("name", [
        "Distrito Nacional",
        "Santo Domingo Este",
        "Santo Domingo Norte",
        "Santo Domingo Oeste",
      ]);

    expect(error).toBeNull();
    expect(data).toHaveLength(4);
  });
});
```

- [ ] **Step 2: Create extensions and enum types**

Create `supabase/migrations/0001_extensions.sql`:

```sql
create extension if not exists pgcrypto;
create extension if not exists citext;

create type public.app_role as enum (
  'user', 'agent_pending', 'agent', 'agency_supervisor', 'agency_admin', 'admin'
);
create type public.property_operation as enum ('rent', 'sale');
create type public.property_type as enum ('apartment', 'house', 'room', 'studio', 'penthouse', 'villa');
create type public.property_status as enum (
  'draft', 'in_review', 'changes_requested', 'published', 'confirmation_due',
  'paused', 'reserved', 'rented', 'sold', 'rejected', 'archived'
);
create type public.location_visibility as enum ('exact', 'approximate', 'sector_only');
```

- [ ] **Step 3: Create identity and location tables**

Create `supabase/migrations/0002_identity_and_locations.sql`:

```sql
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  role public.app_role not null default 'user',
  full_name text not null default '',
  phone text,
  avatar_url text,
  suspended_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.agent_profiles (
  user_id uuid primary key references public.profiles(id) on delete cascade,
  slug citext unique not null,
  professional_name text not null,
  biography text not null default '',
  whatsapp_phone text not null,
  instagram_url text,
  facebook_url text,
  approval_status text not null check (approval_status in ('pending','approved','rejected')) default 'pending',
  trust_score integer not null default 0 check (trust_score between 0 and 100),
  approved_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.locations (
  id uuid primary key default gen_random_uuid(),
  parent_id uuid references public.locations(id) on delete restrict,
  kind text not null check (kind in ('province','municipality','sector')),
  name text not null,
  slug citext not null,
  active boolean not null default true,
  sort_order integer not null default 0,
  unique(parent_id, slug)
);

create index locations_parent_kind_idx on public.locations(parent_id, kind, active);
```

- [ ] **Step 4: Create the property tables and publication invariants**

Create `supabase/migrations/0003_properties.sql`:

```sql
create table public.properties (
  id uuid primary key default gen_random_uuid(),
  code text unique not null,
  agent_id uuid not null references public.agent_profiles(user_id) on delete restrict,
  operation public.property_operation not null,
  property_type public.property_type not null,
  status public.property_status not null default 'draft',
  title text not null default '',
  description text not null default '',
  price_dop numeric(14,2),
  bedrooms smallint not null default 0 check (bedrooms >= 0),
  bathrooms numeric(4,1) not null default 0 check (bathrooms >= 0),
  parking_spaces smallint not null default 0 check (parking_spaces >= 0),
  furnished boolean not null default false,
  construction_m2 numeric(10,2),
  land_m2 numeric(10,2),
  floor_number smallint,
  province_id uuid not null references public.locations(id),
  municipality_id uuid not null references public.locations(id),
  sector_id uuid not null references public.locations(id),
  public_address text,
  private_address text not null default '',
  latitude numeric(9,6),
  longitude numeric(9,6),
  location_visibility public.location_visibility not null default 'approximate',
  requirements text not null default '',
  included_services text[] not null default '{}',
  cover_media_id uuid,
  view_count bigint not null default 0 check (view_count >= 0),
  verification_level smallint not null default 0 check (verification_level between 0 and 5),
  is_featured boolean not null default false,
  published_at timestamptz,
  next_confirmation_at timestamptz,
  confirmed_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint property_publishable_fields check (
    status not in ('published','confirmation_due','reserved','rented','sold')
    or (price_dop is not null and price_dop > 0 and title <> '' and description <> '')
  )
);

create table public.property_media (
  id uuid primary key default gen_random_uuid(),
  property_id uuid not null references public.properties(id) on delete cascade,
  storage_path text not null,
  alt_text text not null default '',
  sort_order integer not null default 0,
  width integer,
  height integer,
  created_at timestamptz not null default now(),
  unique(property_id, storage_path)
);

alter table public.properties
  add constraint properties_cover_media_fk
  foreign key (cover_media_id) references public.property_media(id) on delete set null;

create table public.property_features (
  property_id uuid not null references public.properties(id) on delete cascade,
  feature_key text not null,
  feature_value text not null default 'true',
  primary key(property_id, feature_key)
);

create index properties_public_search_idx
  on public.properties(status, operation, province_id, municipality_id, sector_id, property_type, price_dop);
create index properties_popular_idx on public.properties(status, view_count desc, published_at desc);
create index properties_verified_idx on public.properties(status, verification_level desc, published_at desc);
create index property_media_order_idx on public.property_media(property_id, sort_order);
```

- [ ] **Step 5: Seed the first territory tree**

Create `supabase/seed.sql` with deterministic UUIDs:

```sql
insert into public.locations (id, parent_id, kind, name, slug, sort_order) values
('10000000-0000-0000-0000-000000000001', null, 'province', 'Distrito Nacional', 'distrito-nacional', 1),
('10000000-0000-0000-0000-000000000002', null, 'province', 'Santo Domingo', 'santo-domingo', 2),
('20000000-0000-0000-0000-000000000001', '10000000-0000-0000-0000-000000000001', 'municipality', 'Distrito Nacional', 'distrito-nacional', 1),
('20000000-0000-0000-0000-000000000002', '10000000-0000-0000-0000-000000000002', 'municipality', 'Santo Domingo Este', 'santo-domingo-este', 1),
('20000000-0000-0000-0000-000000000003', '10000000-0000-0000-0000-000000000002', 'municipality', 'Santo Domingo Norte', 'santo-domingo-norte', 2),
('20000000-0000-0000-0000-000000000004', '10000000-0000-0000-0000-000000000002', 'municipality', 'Santo Domingo Oeste', 'santo-domingo-oeste', 3),
('30000000-0000-0000-0000-000000000001', '20000000-0000-0000-0000-000000000001', 'sector', 'Piantini', 'piantini', 1),
('30000000-0000-0000-0000-000000000002', '20000000-0000-0000-0000-000000000001', 'sector', 'Naco', 'naco', 2),
('30000000-0000-0000-0000-000000000003', '20000000-0000-0000-0000-000000000002', 'sector', 'Alma Rosa', 'alma-rosa', 1),
('30000000-0000-0000-0000-000000000004', '20000000-0000-0000-0000-000000000002', 'sector', 'Los Mina', 'los-mina', 2),
('30000000-0000-0000-0000-000000000005', '20000000-0000-0000-0000-000000000003', 'sector', 'Villa Mella', 'villa-mella', 1),
('30000000-0000-0000-0000-000000000006', '20000000-0000-0000-0000-000000000004', 'sector', 'Herrera', 'herrera', 1)
on conflict do nothing;
```

- [ ] **Step 6: Reset the database, generate types, and run the schema test**

Run:

```bash
pnpm supabase start
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/integration/schema.test.ts
```

Expected: PASS and four target territory names are returned.

- [ ] **Step 7: Commit**

```bash
git add supabase src/types/database.ts tests/integration/schema.test.ts
git commit -m "feat: add core property database schema"
```

---

### Task 3: Supabase Clients, Authentication, and Role Authorization

**Files:**
- Create: `src/lib/supabase/browser.ts`
- Create: `src/lib/supabase/server.ts`
- Create: `src/lib/supabase/admin.ts`
- Create: `src/lib/supabase/middleware.ts`
- Create: `src/lib/auth/authorization.ts`
- Create: `src/middleware.ts`
- Create: `src/features/auth/actions.ts`
- Create: `src/app/(public)/auth/login/page.tsx`
- Create: `src/app/(public)/auth/registro/page.tsx`
- Create: `src/app/(public)/auth/callback/route.ts`
- Create: `tests/unit/authorization.test.ts`

**Interfaces:**
- Consumes: `profiles.role` and Supabase Auth.
- Produces: `requireUser()`, `requireRole(allowedRoles)`, `signUpAgent(input)`, `signIn(input)`, protected route middleware.

- [ ] **Step 1: Write failing authorization tests**

Create `tests/unit/authorization.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { canAccessProfessionalPanel, canPublishProperty } from "@/lib/auth/authorization";

describe("authorization", () => {
  it("allows pending agents into the professional panel", () => {
    expect(canAccessProfessionalPanel("agent_pending")).toBe(true);
  });

  it("blocks pending agents from publishing", () => {
    expect(canPublishProperty("agent_pending", false)).toBe(false);
  });

  it("allows approved agents with an active plan to publish", () => {
    expect(canPublishProperty("agent", true)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests and verify failure**

Run:

```bash
pnpm test tests/unit/authorization.test.ts
```

Expected: FAIL because authorization functions do not exist.

- [ ] **Step 3: Implement pure authorization rules**

Create `src/lib/auth/authorization.ts`:

```ts
import type { AppRole } from "@/features/auth/types";

export const professionalRoles: AppRole[] = [
  "agent_pending",
  "agent",
  "agency_supervisor",
  "agency_admin",
  "admin",
];

export function canAccessProfessionalPanel(role: AppRole): boolean {
  return professionalRoles.includes(role);
}

export function canPublishProperty(role: AppRole, hasActivePlan: boolean): boolean {
  return ["agent", "agency_supervisor", "agency_admin", "admin"].includes(role) && hasActivePlan;
}

export function canAccessAdmin(role: AppRole): boolean {
  return role === "admin";
}
```

Create `src/features/auth/types.ts`:

```ts
export type AppRole =
  | "user"
  | "agent_pending"
  | "agent"
  | "agency_supervisor"
  | "agency_admin"
  | "admin";
```

- [ ] **Step 4: Implement Supabase clients and middleware**

Create server/browser/admin clients using `@supabase/ssr`. The server client must use cookies from `next/headers`; the admin client must use `SUPABASE_SERVICE_ROLE_KEY` and `persistSession: false`. `src/middleware.ts` must refresh auth and redirect unauthenticated `/panel` or `/admin` requests to `/auth/login`.

The exported signatures must be:

```ts
export function createBrowserClient(): SupabaseClient<Database>;
export async function createServerClient(): Promise<SupabaseClient<Database>>;
export function createAdminClient(): SupabaseClient<Database>;
export async function updateSession(request: NextRequest): Promise<NextResponse>;
```

- [ ] **Step 5: Implement registration and login actions**

Create `src/features/auth/actions.ts` with these schemas and signatures:

```ts
const agentRegistrationSchema = z.object({
  email: z.string().email(),
  password: z.string().min(10),
  fullName: z.string().min(3).max(100),
  professionalName: z.string().min(3).max(100),
  slug: z.string().regex(/^[a-z0-9-]+$/),
  whatsappPhone: z.string().regex(/^1?8\d{9}$/),
});

export async function signUpAgent(input: z.infer<typeof agentRegistrationSchema>): Promise<ActionResult<{ userId: string }>>;
export async function signIn(input: { email: string; password: string }): Promise<ActionResult<{ redirectTo: string }>>;
export async function signOut(): Promise<void>;
```

After a successful sign-up, insert `profiles.role = 'agent_pending'` and an `agent_profiles` row with `approval_status = 'pending'`.

- [ ] **Step 6: Create minimal login and registration pages**

Build accessible forms using React Hook Form and Zod. Registration success copy must state: “Tu cuenta fue creada. Completa la verificación; publicar permanecerá bloqueado hasta la aprobación de Alqui-RD.”

- [ ] **Step 7: Run tests and verify**

Run:

```bash
pnpm test tests/unit/authorization.test.ts
pnpm typecheck
pnpm build
```

Expected: all commands exit with status 0.

- [ ] **Step 8: Commit**

```bash
git add src tests/unit/authorization.test.ts
git commit -m "feat: add authentication and role authorization"
```

---

### Task 4: Row-Level Security, Storage Policies, and Audit Logging

**Files:**
- Create: `supabase/migrations/0004_audit_core_rls.sql`
- Create: `src/features/audit/service.ts`
- Create: `tests/integration/rls.test.ts`

**Interfaces:**
- Consumes: authenticated user ID, profile role, property ownership.
- Produces: RLS policies and `writeAuditLog(input)`.

- [ ] **Step 1: Add a failing private-address RLS test**

Create `tests/integration/rls.test.ts` that authenticates as a regular user, selects a published property through a public RPC named `search_properties`, and asserts the returned object has no `private_address` key.

```ts
expect(result).not.toHaveProperty("private_address");
expect(result).toHaveProperty("location_label");
```

- [ ] **Step 2: Add audit table and helper function**

Create `supabase/migrations/0004_audit_core_rls.sql`:

```sql
create table public.audit_logs (
  id bigint generated always as identity primary key,
  actor_id uuid references public.profiles(id) on delete set null,
  action text not null,
  entity_type text not null,
  entity_id text not null,
  before_data jsonb,
  after_data jsonb,
  metadata jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now()
);

alter table public.audit_logs enable row level security;
create policy "admins read audit logs"
  on public.audit_logs for select
  using ((select role from public.profiles where id = auth.uid()) = 'admin');
```

Create `src/features/audit/service.ts`:

```ts
import { createAdminClient } from "@/lib/supabase/admin";

export type AuditInput = {
  actorId: string | null;
  action: string;
  entityType: string;
  entityId: string;
  beforeData?: unknown;
  afterData?: unknown;
  metadata?: Record<string, unknown>;
};

export async function writeAuditLog(input: AuditInput): Promise<void> {
  const db = createAdminClient();
  const { error } = await db.from("audit_logs").insert({
    actor_id: input.actorId,
    action: input.action,
    entity_type: input.entityType,
    entity_id: input.entityId,
    before_data: input.beforeData ?? null,
    after_data: input.afterData ?? null,
    metadata: input.metadata ?? {},
  });
  if (error) throw new Error(`audit_write_failed:${error.message}`);
}
```

- [ ] **Step 3: Add RLS policies and safe public search RPC**

The migration must enable RLS on `profiles`, `agent_profiles`, `locations`, `properties`, `property_media`, `property_features`, and `audit_logs`. Every later migration must enable RLS for its own new tables. Required core policies:

- Public reads only `properties.status = 'published'`, via RPC that omits private fields.
- Agents read/write only their own draft and workflow records.
- Agency-specific access is added in Task 15 after agency tables and `properties.agency_id` exist.
- Admins can read/write every domain table.
- Identity documents and payment proofs are never public.
- Storage bucket `property-media` is public-read only for paths attached to published properties.
- Storage buckets `verification-documents` and `payment-proofs` remain private.

Create RPC signature:

```sql
create or replace function public.search_properties(
  p_operation public.property_operation default null,
  p_province uuid default null,
  p_municipality uuid default null,
  p_sector uuid default null,
  p_type public.property_type default null,
  p_min_price numeric default null,
  p_max_price numeric default null,
  p_bedrooms smallint default null,
  p_bathrooms numeric default null,
  p_parking smallint default null,
  p_furnished boolean default null,
  p_sort text default 'recent',
  p_limit integer default 24,
  p_offset integer default 0
) returns table (
  id uuid, code text, title text, operation public.property_operation,
  property_type public.property_type, price_dop numeric,
  bedrooms smallint, bathrooms numeric, parking_spaces smallint,
  furnished boolean, municipality_name text, sector_name text,
  location_label text, cover_path text, published_at timestamptz
)
```

Use `security definer`, set `search_path = public`, and never select `private_address`, exact coordinates, phone, or unpublished records.

- [ ] **Step 4: Reset, regenerate types, and run RLS test**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/integration/rls.test.ts
```

Expected: PASS; public payload excludes private address.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/0004_audit_core_rls.sql src/features/audit tests/integration/rls.test.ts src/types/database.ts
git commit -m "feat: enforce row security and audit logging"
```
