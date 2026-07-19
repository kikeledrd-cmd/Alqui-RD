# Ola 2 — Supply Engine

### Task 5: Location Catalog and Public Search Domain

**Files:**
- Create: `src/features/locations/queries.ts`
- Create: `src/features/properties/search-schema.ts`
- Create: `src/features/properties/search-service.ts`
- Create: `src/components/search/property-search-form.tsx`
- Create: `src/components/property/property-card.tsx`
- Create: `src/app/(public)/buscar/page.tsx`
- Create: `tests/unit/property-search-schema.test.ts`

**Interfaces:**
- Consumes: `search_properties` RPC and `locations` rows.
- Produces: `PropertySearchInput`, `searchProperties(input)`, `getLocationChildren(parentId, kind)`.

- [ ] **Step 1: Write failing search schema tests**

Create `tests/unit/property-search-schema.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { propertySearchSchema } from "@/features/properties/search-schema";

describe("propertySearchSchema", () => {
  it("accepts a rent search with price range", () => {
    const result = propertySearchSchema.parse({
      operation: "rent",
      minPrice: "15000",
      maxPrice: "35000",
      page: "1",
    });
    expect(result.minPrice).toBe(15000);
    expect(result.maxPrice).toBe(35000);
  });

  it("rejects an inverted price range", () => {
    expect(() => propertySearchSchema.parse({ minPrice: "40000", maxPrice: "20000" })).toThrow();
  });
});
```

- [ ] **Step 2: Implement the schema**

Create `src/features/properties/search-schema.ts`:

```ts
import { z } from "zod";

const optionalNumber = z.preprocess(
  (value) => (value === "" || value === undefined ? undefined : Number(value)),
  z.number().nonnegative().optional(),
);

export const propertySearchSchema = z
  .object({
    operation: z.enum(["rent", "sale"]).optional(),
    provinceId: z.string().uuid().optional(),
    municipalityId: z.string().uuid().optional(),
    sectorId: z.string().uuid().optional(),
    propertyType: z.enum(["apartment", "house", "room", "studio", "penthouse", "villa"]).optional(),
    minPrice: optionalNumber,
    maxPrice: optionalNumber,
    bedrooms: optionalNumber,
    bathrooms: optionalNumber,
    parking: optionalNumber,
    furnished: z.preprocess((v) => (v === "true" ? true : v === "false" ? false : undefined), z.boolean().optional()),
    sort: z.enum(["recent", "price_asc", "price_desc", "popular", "verified"]).default("recent"),
    page: z.preprocess((v) => Number(v ?? 1), z.number().int().min(1)).default(1),
  })
  .refine((v) => v.minPrice === undefined || v.maxPrice === undefined || v.minPrice <= v.maxPrice, {
    message: "El precio mínimo no puede superar el máximo",
    path: ["maxPrice"],
  });

export type PropertySearchInput = z.infer<typeof propertySearchSchema>;
```

- [ ] **Step 3: Implement location and search queries**

Create `searchProperties(input)` to call `search_properties` with `limit = 24`, `offset = (page - 1) * 24`. Apply sorting in SQL/RPC rather than in the browser. Return:

```ts
export type PropertySearchResult = {
  items: PublicPropertyCard[];
  page: number;
  pageSize: 24;
  hasNextPage: boolean;
};
```

`getLocationChildren(parentId, kind)` must return active locations ordered by `sort_order,name`.

- [ ] **Step 4: Build the search page and no-results recovery**

`/buscar` must:

- Parse `searchParams` with `propertySearchSchema.safeParse`.
- Render filters as GET parameters.
- Render 24 cards per page.
- When empty, show three links: remove sector, increase max price by 20%, and show all property types.
- Preserve current filters in pagination links.

- [ ] **Step 5: Run tests and build**

Run:

```bash
pnpm test tests/unit/property-search-schema.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/features/locations src/features/properties src/components/search src/components/property src/app/'(public)'/buscar tests/unit/property-search-schema.test.ts
git commit -m "feat: add location filters and public property search"
```

---

### Task 6: Agent Profile Approval and Active Plan Gate

**Files:**
- Create: `supabase/migrations/0005_agent_verification_billing.sql`
- Create: `src/features/billing/types.ts`
- Create: `src/features/billing/queries.ts`
- Create: `src/features/verifications/agent-actions.ts`
- Create: `src/app/(professional)/panel/verificacion/page.tsx`
- Create: `src/features/auth/guards.ts`
- Create: `src/features/moderation/agent-actions.ts`
- Create: `src/app/(admin)/admin/agentes/page.tsx`
- Create: `tests/unit/publish-gate.test.ts`

**Interfaces:**
- Consumes: profile role, agent approval status, subscription dates.
- Produces: `getPublishEligibility(userId)`, `approveAgent(input)`, `rejectAgent(input)`.

- [ ] **Step 1: Create billing tables and a trial plan**

In `supabase/migrations/0005_agent_verification_billing.sql` create:

```sql
create table public.subscription_plans (
  id uuid primary key default gen_random_uuid(),
  code citext unique not null,
  name text not null,
  audience text not null check (audience in ('agent','agency')),
  property_limit integer not null check (property_limit >= 0),
  monthly_price_dop numeric(12,2) not null check (monthly_price_dop >= 0),
  included_promotions integer not null default 0,
  active boolean not null default true,
  created_at timestamptz not null default now()
);

create table public.subscriptions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  plan_id uuid not null references public.subscription_plans(id),
  status text not null check (status in ('trial','active','past_due','expired','cancelled')),
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  created_at timestamptz not null default now(),
  check (ends_at > starts_at)
);

create table public.verification_requests (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.profiles(id) on delete cascade,
  kind text not null check (kind in ('identity','professional_reference')),
  document_path text not null,
  status text not null check (status in ('submitted','approved','rejected')) default 'submitted',
  rejection_reason text,
  reviewed_by uuid references public.profiles(id),
  reviewed_at timestamptz,
  created_at timestamptz not null default now()
);

alter table public.subscription_plans enable row level security;
alter table public.subscriptions enable row level security;
alter table public.verification_requests enable row level security;

insert into public.subscription_plans (code,name,audience,property_limit,monthly_price_dop,included_promotions)
values ('trial-agent','Prueba Agente','agent',3,0,0)
on conflict (code) do nothing;
```

- [ ] **Step 2: Write failing publish-gate tests**

```ts
import { describe, expect, it } from "vitest";
import { evaluatePublishEligibility } from "@/features/auth/guards";

describe("publish eligibility", () => {
  it("blocks an unapproved agent", () => {
    expect(evaluatePublishEligibility({ role: "agent_pending", approved: false, subscriptionActive: true, activeProperties: 0, limit: 3 })).toEqual({ allowed: false, reason: "agent_not_approved" });
  });

  it("blocks an agent at the plan limit", () => {
    expect(evaluatePublishEligibility({ role: "agent", approved: true, subscriptionActive: true, activeProperties: 3, limit: 3 })).toEqual({ allowed: false, reason: "property_limit_reached" });
  });
});
```

- [ ] **Step 3: Implement the pure gate and database query**

```ts
export type PublishEligibilityInput = {
  role: AppRole;
  approved: boolean;
  subscriptionActive: boolean;
  activeProperties: number;
  limit: number;
};

export function evaluatePublishEligibility(input: PublishEligibilityInput):
  | { allowed: true }
  | { allowed: false; reason: "agent_not_approved" | "no_active_plan" | "property_limit_reached" | "account_suspended" };

export async function getPublishEligibility(userId: string): Promise<ReturnType<typeof evaluatePublishEligibility>>;
```

Count property statuses `published`, `in_review`, `changes_requested`, `confirmation_due`, `paused`, and `reserved` against the plan limit. Exclude `draft`, `rented`, `sold`, `rejected`, and `archived`.

- [ ] **Step 4: Implement private agent-document submission**

`submitAgentVerificationDocument({ kind, file })` must accept PDF/JPEG/PNG up to 8 MB, upload to `verification-documents/<userId>/<uuid>`, and insert `verification_requests.status = 'submitted'`. Agents can list their own request status but never receive a reusable raw storage path; admins view documents only through short-lived signed URLs. The verification page must clearly show that approval is pending and publishing remains blocked.

- [ ] **Step 5: Implement agent approval actions**

`approveAgent({ agentId, adminId })` must reject approval when no submitted or approved `identity` verification request exists, then:

1. Set `agent_profiles.approval_status = 'approved'` and `approved_at = now()`.
2. Set `profiles.role = 'agent'`.
3. Create a 30-day trial subscription if the agent has no subscription.
4. Create an audit log with action `agent.approved`.

`rejectAgent({ agentId, adminId, reason })` must preserve the account, set `approval_status = 'rejected'`, and log `agent.rejected`.

- [ ] **Step 6: Build the admin queue**

`/admin/agentes` must list pending agents with full name, professional name, phone, registration date, and actions “Aprobar” and “Rechazar”. Require `admin` role in the layout.

- [ ] **Step 7: Run tests and reset schema**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/unit/publish-gate.test.ts
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add supabase/migrations/0005_agent_verification_billing.sql src/features/billing src/features/verifications/agent-actions.ts src/features/auth/guards.ts src/features/moderation src/app/'(professional)'/panel/verificacion src/app/'(admin)' tests/unit/publish-gate.test.ts src/types/database.ts
git commit -m "feat: add agent approval and publishing eligibility"
```

---

### Task 7: Property Wizard, State Machine, and Draft Persistence

**Files:**
- Create: `src/features/properties/types.ts`
- Create: `src/features/properties/property-schema.ts`
- Create: `src/features/properties/state-machine.ts`
- Create: `src/features/properties/actions.ts`
- Create: `src/components/property/property-wizard.tsx`
- Create: `src/app/(professional)/panel/propiedades/nueva/page.tsx`
- Create: `src/app/(professional)/panel/propiedades/[id]/editar/page.tsx`
- Create: `tests/unit/property-state-machine.test.ts`
- Create: `tests/unit/property-schema.test.ts`

**Interfaces:**
- Consumes: `getPublishEligibility`, current agent ID, location IDs.
- Produces: `savePropertyDraft(input)`, `submitPropertyForReview(propertyId)`, `transitionPropertyStatus(input)`.

- [ ] **Step 1: Write failing state transition tests**

```ts
import { describe, expect, it } from "vitest";
import { canTransitionProperty } from "@/features/properties/state-machine";

describe("property state machine", () => {
  it("allows draft to in_review", () => {
    expect(canTransitionProperty("draft", "in_review", "agent")).toBe(true);
  });

  it("blocks agent from publishing directly", () => {
    expect(canTransitionProperty("in_review", "published", "agent")).toBe(false);
  });

  it("allows admin to approve a reviewed property", () => {
    expect(canTransitionProperty("in_review", "published", "admin")).toBe(true);
  });
});
```

- [ ] **Step 2: Implement explicit transition table**

```ts
const transitions: Record<PropertyStatus, Partial<Record<PropertyStatus, AppRole[]>>> = {
  draft: { in_review: ["agent", "agency_supervisor", "agency_admin", "admin"], archived: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  in_review: { changes_requested: ["admin"], published: ["admin"], rejected: ["admin"] },
  changes_requested: { in_review: ["agent", "agency_supervisor", "agency_admin", "admin"], archived: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  published: { confirmation_due: ["admin"], paused: ["agent", "agency_supervisor", "agency_admin", "admin"], reserved: ["agent", "agency_supervisor", "agency_admin", "admin"], rented: ["agent", "agency_supervisor", "agency_admin", "admin"], sold: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  confirmation_due: { published: ["agent", "agency_supervisor", "agency_admin", "admin"], paused: ["admin"] },
  paused: { published: ["agent", "agency_supervisor", "agency_admin", "admin"], archived: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  reserved: { published: ["agent", "agency_supervisor", "agency_admin", "admin"], rented: ["agent", "agency_supervisor", "agency_admin", "admin"], sold: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  rented: { archived: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  sold: { archived: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  rejected: { draft: ["agent", "agency_supervisor", "agency_admin", "admin"], archived: ["agent", "agency_supervisor", "agency_admin", "admin"] },
  archived: {},
};
```

Export `canTransitionProperty(from, to, role)` and make all status changes pass through it.

- [ ] **Step 3: Implement the six-step validation schema**

`propertyDraftSchema` must validate operation, type, price, location IDs, private address, visibility, characteristics, title, description, requirements, services, and media IDs. `submitPropertySchema` must additionally require:

- `priceDop > 0`
- title at least 12 characters
- description at least 80 characters
- valid province/municipality/sector relationship
- at least 3 media records
- exactly one cover media

For `room`, allow `constructionM2` to be absent. For `villa`, require either `constructionM2` or `landM2`.

- [ ] **Step 4: Implement draft save and submit actions**

```ts
export async function savePropertyDraft(input: PropertyDraftInput): Promise<ActionResult<{ propertyId: string; code: string }>>;
export async function submitPropertyForReview(propertyId: string): Promise<ActionResult<{ status: "in_review" }>>;
export async function transitionPropertyStatus(input: { propertyId: string; to: PropertyStatus; reason?: string }): Promise<ActionResult<{ status: PropertyStatus }>>;
```

Generate property codes with database sequence format `ARD-000001`. Every transition creates an audit record. Draft saves must use upsert and return field errors without deleting prior data.

- [ ] **Step 5: Build the wizard with local recovery**

The client component must:

- Save each step to the server and mirror the latest valid draft in `localStorage` key `alquird-property-draft:<propertyId>`.
- Restore unsaved form values after a network failure.
- Show a persistent “Guardado” timestamp.
- Prevent step 6 submission until schema requirements pass.
- Never render `privateAddress` outside professional/admin routes.

- [ ] **Step 6: Run tests and build**

Run:

```bash
pnpm test tests/unit/property-state-machine.test.ts tests/unit/property-schema.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/features/properties src/components/property/property-wizard.tsx src/app/'(professional)' tests/unit/property-state-machine.test.ts tests/unit/property-schema.test.ts
git commit -m "feat: add resilient property publishing wizard"
```

---

### Task 8: Property Media Upload, Ordering, and Safe Storage

**Files:**
- Create: `src/features/media/validation.ts`
- Create: `src/features/media/service.ts`
- Create: `src/app/api/media/property/route.ts`
- Create: `src/components/property/property-media-uploader.tsx`
- Create: `tests/unit/media-validation.test.ts`

**Interfaces:**
- Consumes: authenticated agent, property ownership, storage bucket `property-media`.
- Produces: `validatePropertyImage(file)`, `uploadPropertyImage(input)`, `reorderPropertyMedia(input)`, `deletePropertyMedia(input)`.

- [ ] **Step 1: Write failing validation tests**

```ts
import { describe, expect, it } from "vitest";
import { validatePropertyImageMetadata } from "@/features/media/validation";

describe("property image validation", () => {
  it("accepts jpeg under 8 MB", () => {
    expect(validatePropertyImageMetadata({ type: "image/jpeg", size: 2_000_000, width: 1600, height: 1200 })).toEqual({ ok: true });
  });

  it("rejects tiny images", () => {
    expect(validatePropertyImageMetadata({ type: "image/jpeg", size: 200_000, width: 400, height: 300 })).toEqual({ ok: false, reason: "dimensions_too_small" });
  });
});
```

- [ ] **Step 2: Implement validation rules**

Accept JPEG, PNG, or WebP, maximum 8 MB, minimum 1000×750, maximum 30 images per property. Strip EXIF metadata client-side by decoding and re-encoding to WebP quality 0.82 before upload.

- [ ] **Step 3: Implement signed upload flow**

`POST /api/media/property` accepts `{ propertyId, filename, contentType }`, verifies ownership and plan access, and returns a signed upload URL with path:

```text
agents/<agentId>/properties/<propertyId>/<uuid>.webp
```

After client upload, call `finalizePropertyMedia({ propertyId, storagePath, width, height, altText })`. The server verifies the object exists before inserting `property_media`.

- [ ] **Step 4: Implement ordering and cover selection transaction**

```ts
export async function reorderPropertyMedia(input: {
  propertyId: string;
  orderedMediaIds: string[];
  coverMediaId: string;
}): Promise<ActionResult<{ count: number }>>;
```

Validate that all media IDs belong to the property, update `sort_order`, then set `properties.cover_media_id` in one transaction/RPC.

- [ ] **Step 5: Build uploader recovery UI**

Display per-file states `queued`, `compressing`, `uploading`, `complete`, `failed`. Failed items must retain their local file reference and expose “Reintentar”. The rest of the wizard must remain usable when one upload fails.

- [ ] **Step 6: Run tests and build**

Run:

```bash
pnpm test tests/unit/media-validation.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/features/media src/app/api/media src/components/property/property-media-uploader.tsx tests/unit/media-validation.test.ts
git commit -m "feat: add secure property media workflow"
```

---

### Task 9: Admin Property Moderation and Trust Progression

**Files:**
- Create: `src/features/moderation/property-actions.ts`
- Create: `src/features/moderation/trust-score.ts`
- Create: `src/app/(admin)/admin/propiedades/page.tsx`
- Create: `src/app/(admin)/admin/propiedades/[id]/page.tsx`
- Create: `tests/unit/trust-score.test.ts`

**Interfaces:**
- Consumes: property state machine, audit logger.
- Produces: `approveProperty`, `requestPropertyChanges`, `rejectProperty`, `calculateTrustScore`.

- [ ] **Step 1: Write failing trust score tests**

```ts
import { describe, expect, it } from "vitest";
import { calculateTrustScore } from "@/features/moderation/trust-score";

describe("trust score", () => {
  it("rewards complete accurate history", () => {
    expect(calculateTrustScore({ approved: 10, changesRequested: 1, rejected: 0, expiredUnconfirmed: 0, unresolvedReports: 0 })).toBe(94);
  });

  it("never drops below zero", () => {
    expect(calculateTrustScore({ approved: 0, changesRequested: 10, rejected: 10, expiredUnconfirmed: 10, unresolvedReports: 10 })).toBe(0);
  });
});
```

- [ ] **Step 2: Implement deterministic scoring**

Use:

```ts
score = 50
  + min(approved * 5, 40)
  - changesRequested * 3
  - rejected * 10
  - expiredUnconfirmed * 4
  - unresolvedReports * 15;
```

Clamp to 0–100. Auto-publish eligibility requires score at least 90, at least 10 approved properties, zero unresolved reports, and an admin-enabled `auto_publish_enabled` flag.

- [ ] **Step 3: Implement moderation actions**

`approveProperty` must validate publication invariants, set `published_at`, calculate `next_confirmation_at` as +15 days for rent or +30 days for sale, and log `property.approved`.

`requestPropertyChanges` requires at least one reason from this exact set:

```ts
"incomplete_information" | "poor_images" | "suspicious_price" | "location_inconsistent" | "duplicate" | "misleading_content" | "unresolved_report"
```

`rejectProperty` requires a human-readable explanation of at least 20 characters.

- [ ] **Step 4: Build the moderation queue**

Admin list filters: `in_review`, `changes_requested`, `rejected`, agent, municipality, property type. Detail page must show every public field plus private address, media, owner agent, previous moderation events, and three actions.

- [ ] **Step 5: Run tests and build**

Run:

```bash
pnpm test tests/unit/trust-score.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/features/moderation src/app/'(admin)'/admin/propiedades tests/unit/trust-score.test.ts
git commit -m "feat: add property moderation and trust scoring"
```

---

### Task 10: Public Home, Property Detail, Carousel, and Agent Mini Office

**Files:**
- Create: `src/app/(public)/page.tsx`
- Create: `src/app/(public)/inmueble/[code]/page.tsx`
- Create: `src/app/(public)/agente/[slug]/page.tsx`
- Create: `src/features/properties/public-queries.ts`
- Create: `src/components/property/property-carousel.tsx`
- Create: `src/components/property/property-facts.tsx`
- Create: `src/components/property/location-map.tsx`
- Create: `tests/unit/public-location.test.ts`

**Interfaces:**
- Consumes: safe public property views/RPC.
- Produces: `getPublicPropertyByCode(code)`, `getPublicAgentBySlug(slug)`, `formatPublicLocation(property)`.

- [ ] **Step 1: Write failing location privacy tests**

```ts
import { describe, expect, it } from "vitest";
import { formatPublicLocation } from "@/features/properties/public-location";

describe("public property location", () => {
  it("shows only sector for sector_only", () => {
    expect(formatPublicLocation({ visibility: "sector_only", sector: "Alma Rosa", municipality: "Santo Domingo Este", publicAddress: "Calle 1" })).toEqual({ label: "Alma Rosa, Santo Domingo Este", showMap: false });
  });

  it("never returns the private address", () => {
    const result = formatPublicLocation({ visibility: "approximate", sector: "Naco", municipality: "Distrito Nacional", publicAddress: null });
    expect(JSON.stringify(result)).not.toContain("private");
  });
});
```

- [ ] **Step 2: Implement public queries with cache tags**

Only return published properties. Use cache tags `property:<id>`, `agent:<id>`, and `search`. On moderation changes, invalidate the matching tags.

- [ ] **Step 3: Build the home page**

Sections:

1. Hero with `Alquiler` selected by default.
2. Province/municipality, sector, type, price range.
3. Eight properties where `is_featured = true`; when fewer than eight exist, fill with the highest verification level and newest publication date.
4. Eight recent properties.
5. Popular sectors based on published count.
6. CTA “Publica tus propiedades”.
7. CTA “Tengo una propiedad”.

All search controls must submit a GET request to `/buscar`.

- [ ] **Step 4: Build property detail**

The detail page must include:

- Responsive carousel with keyboard arrows and thumbnails.
- Price formatted as `RD$ 35,000` and `/mes` for rent.
- Feature grid.
- Verification badges.
- Public location according to visibility.
- Agent card.
- Fixed mobile contact bar.
- Similar properties by same operation, municipality, and type, excluding current ID.
- Open Graph metadata using cover image and sanitized description.

- [ ] **Step 5: Build agent mini office**

Show professional name, biography, WhatsApp, social links, zones, badges, calculated response time when at least five answered leads exist, and active listings. Do not show private performance metrics.

- [ ] **Step 6: Run tests and build**

Run:

```bash
pnpm test tests/unit/public-location.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/app/'(public)' src/features/properties/public-queries.ts src/features/properties/public-location.ts src/components/property tests/unit/public-location.test.ts
git commit -m "feat: add public property and agent experience"
```
