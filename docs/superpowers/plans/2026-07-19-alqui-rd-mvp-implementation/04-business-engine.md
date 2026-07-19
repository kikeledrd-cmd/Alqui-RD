# Ola 4 — Business Engine

### Task 14: Manual Payments, Plan Activation, and Promotion Extras

**Files:**
- Create: `supabase/migrations/0007_payments_promotions.sql`
- Create: `src/features/billing/payment-provider.ts`
- Create: `src/features/billing/manual-payment.ts`
- Create: `src/features/billing/actions.ts`
- Create: `src/app/(professional)/panel/plan/page.tsx`
- Create: `src/app/(admin)/admin/pagos/page.tsx`
- Create: `tests/unit/manual-payment.test.ts`

**Interfaces:**
- Consumes: plans, subscriptions, private storage `payment-proofs`.
- Produces: `PaymentProvider`, `submitManualPayment`, `approveManualPayment`, `rejectManualPayment`.

- [ ] **Step 1: Add payment and promotion schema**

Create `supabase/migrations/0007_payments_promotions.sql`:

```sql
create table public.payments (
  id uuid primary key default gen_random_uuid(),
  payer_user_id uuid not null references public.profiles(id),
  subscription_id uuid references public.subscriptions(id),
  amount_dop numeric(12,2) not null check (amount_dop > 0),
  method text not null check (method in ('bank_transfer')),
  status text not null check (status in ('submitted','approved','rejected')) default 'submitted',
  proof_path text not null,
  reference text,
  rejection_reason text,
  reviewed_by uuid references public.profiles(id),
  reviewed_at timestamptz,
  created_at timestamptz not null default now()
);

create table public.promotions (
  id uuid primary key default gen_random_uuid(),
  property_id uuid not null references public.properties(id) on delete cascade,
  kind text not null check (kind in ('featured','search_boost')),
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  status text not null check (status in ('pending','active','expired','cancelled')),
  payment_id uuid references public.payments(id),
  check (ends_at > starts_at)
);

alter table public.payments enable row level security;
alter table public.promotions enable row level security;
```

Add policies so payers read their own payments, agency admins read agency payments, and only Alqui-RD admins approve/reject. Proof paths never appear in public responses.

- [ ] **Step 2: Write failing payment activation test**

```ts
import { describe, expect, it } from "vitest";
import { activationPeriod } from "@/features/billing/manual-payment";

describe("manual payment activation", () => {
  it("activates a monthly plan for 30 days", () => {
    expect(activationPeriod(new Date("2026-07-19T12:00:00Z"))).toEqual({
      startsAt: "2026-07-19T12:00:00.000Z",
      endsAt: "2026-08-18T12:00:00.000Z",
    });
  });
});
```

- [ ] **Step 3: Create payment provider abstraction**

```ts
export interface PaymentProvider {
  createCheckout(input: { userId: string; planId: string; returnUrl: string }): Promise<{ enabled: boolean; checkoutUrl?: string }>;
}

export class DisabledAutomaticPaymentProvider implements PaymentProvider {
  async createCheckout(): Promise<{ enabled: false }> {
    return { enabled: false };
  }
}
```

The UI must not show a card-payment button while `enabled` is false.

- [ ] **Step 4: Implement manual payment workflow**

`submitManualPayment` validates proof MIME type PDF/JPEG/PNG, maximum 8 MB, uploads to private bucket, and inserts status `submitted` without activating anything.

`approveManualPayment` runs in one transaction:

1. Set payment `approved`, reviewer, timestamp.
2. Create or extend subscription by 30 days.
3. Activate related promotion when present and set `properties.is_featured = true` for `featured` promotions.
4. Audit `payment.approved`.

`rejectManualPayment` requires reason at least 15 characters and leaves subscription unchanged. The daily renewal job must also expire promotions whose `ends_at <= now()` and clear `properties.is_featured` when no active featured promotion remains.

- [ ] **Step 5: Build professional and admin screens**

Professional page shows current plan, property usage, expiry, bank instructions, upload form, and payment history. Admin page shows proof in a signed private URL, payer, amount, reference, and approve/reject controls.

- [ ] **Step 6: Run tests and schema reset**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/unit/manual-payment.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add supabase/migrations/0007_payments_promotions.sql src/features/billing src/app/'(professional)'/panel/plan src/app/'(admin)'/admin/pagos tests/unit/manual-payment.test.ts src/types/database.ts
git commit -m "feat: add manual plans and payment approval"
```

---

### Task 15: Agency Accounts, Membership Roles, and Team Assignment

**Files:**
- Create: `supabase/migrations/0008_agencies.sql`
- Create: `src/features/agencies/types.ts`
- Create: `src/features/agencies/authorization.ts`
- Create: `src/features/agencies/actions.ts`
- Create: `src/app/(professional)/panel/equipo/page.tsx`
- Create: `tests/unit/agency-authorization.test.ts`

**Interfaces:**
- Consumes: agent profiles and subscriptions.
- Produces: `createAgency`, `inviteAgencyMember`, `changeAgencyMemberRole`, `assignPropertyToAgent`, `canManageAgencyMember`.

- [ ] **Step 1: Add agency tables**

Create `supabase/migrations/0008_agencies.sql`:

```sql
create table public.agencies (
  id uuid primary key default gen_random_uuid(),
  slug citext unique not null,
  name text not null,
  logo_url text,
  description text not null default '',
  whatsapp_phone text not null,
  created_by uuid not null references public.profiles(id),
  created_at timestamptz not null default now()
);

create table public.agency_memberships (
  agency_id uuid not null references public.agencies(id) on delete cascade,
  user_id uuid not null references public.profiles(id) on delete cascade,
  role text not null check (role in ('agency_admin','agency_supervisor','agent')),
  status text not null check (status in ('invited','active','suspended')),
  created_at timestamptz not null default now(),
  primary key(agency_id, user_id)
);

alter table public.properties add column agency_id uuid references public.agencies(id) on delete set null;

alter table public.agencies enable row level security;
alter table public.agency_memberships enable row level security;
```

Add policies matching the exact agency role rules in Step 3.

- [ ] **Step 2: Write failing authorization tests**

```ts
import { describe, expect, it } from "vitest";
import { canManageAgencyMember } from "@/features/agencies/authorization";

describe("agency member authorization", () => {
  it("allows agency admin to change supervisor", () => {
    expect(canManageAgencyMember("agency_admin", "agency_supervisor")).toBe(true);
  });

  it("blocks supervisor from removing agency admin", () => {
    expect(canManageAgencyMember("agency_supervisor", "agency_admin")).toBe(false);
  });
});
```

- [ ] **Step 3: Implement agency permissions**

Rules:

- `agency_admin`: invite, suspend, change roles except demoting the last agency admin, assign properties/leads, view aggregate metrics.
- `agency_supervisor`: assign properties/leads to active agents, view team metrics, cannot edit agency billing or agency admins.
- `agent`: manage assigned properties/leads only.

- [ ] **Step 4: Implement actions and team UI**

Invitations target an existing Alqui-RD account email. Accepting invitation changes membership to `active`. Property assignment checks both users belong to the same agency and logs `property.assigned`.

- [ ] **Step 5: Run tests and schema reset**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/unit/agency-authorization.test.ts
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add supabase/migrations/0008_agencies.sql src/features/agencies src/app/'(professional)'/panel/equipo tests/unit/agency-authorization.test.ts src/types/database.ts
git commit -m "feat: add agency teams and assignment permissions"
```

---

### Task 16: Owner Property Submissions and Ranked Agent Recommendation

**Files:**
- Create: `supabase/migrations/0009_owner_submissions.sql`
- Create: `src/features/owner-submissions/schema.ts`
- Create: `src/features/owner-submissions/recommendation.ts`
- Create: `src/features/owner-submissions/actions.ts`
- Create: `src/app/(public)/tengo-una-propiedad/page.tsx`
- Create: `src/app/(admin)/admin/propietarios/page.tsx`
- Create: `src/app/(admin)/admin/propietarios/[id]/page.tsx`
- Create: `tests/unit/agent-recommendation.test.ts`

**Interfaces:**
- Consumes: agent zones, trust score, response metrics, open lead count, plan tier.
- Produces: `submitOwnerProperty`, `rankAgentCandidates`, `assignOwnerSubmission`.

- [ ] **Step 1: Create owner submission tables**

Create `supabase/migrations/0009_owner_submissions.sql`:

```sql
create table public.owner_submissions (
  id uuid primary key default gen_random_uuid(),
  owner_name text not null,
  phone text not null,
  email text,
  operation public.property_operation not null,
  property_type public.property_type not null,
  expected_price_dop numeric(14,2),
  province_id uuid not null references public.locations(id),
  municipality_id uuid not null references public.locations(id),
  sector_id uuid not null references public.locations(id),
  private_address text not null,
  description text not null,
  status text not null check (status in ('new','validating','assigned','rejected','converted')) default 'new',
  assigned_agent_id uuid references public.agent_profiles(user_id),
  consent_at timestamptz not null,
  created_at timestamptz not null default now()
);

create table public.owner_submission_media (
  id uuid primary key default gen_random_uuid(),
  owner_submission_id uuid not null references public.owner_submissions(id) on delete cascade,
  storage_path text not null,
  sort_order integer not null default 0,
  created_at timestamptz not null default now()
);

create table public.assignment_recommendations (
  id uuid primary key default gen_random_uuid(),
  owner_submission_id uuid not null references public.owner_submissions(id) on delete cascade,
  agent_id uuid not null references public.agent_profiles(user_id),
  score numeric(6,2) not null,
  reasons jsonb not null,
  created_at timestamptz not null default now(),
  unique(owner_submission_id, agent_id)
);

alter table public.owner_submissions enable row level security;
alter table public.owner_submission_media enable row level security;
alter table public.assignment_recommendations enable row level security;
```

Public visitors submit only through a validated server action; agents cannot browse owner submissions; admins can read and assign them.

- [ ] **Step 2: Write failing recommendation tests**

```ts
import { describe, expect, it } from "vitest";
import { rankAgentCandidates } from "@/features/owner-submissions/recommendation";

describe("rankAgentCandidates", () => {
  it("prioritizes zone fit without letting plan tier dominate", () => {
    const ranked = rankAgentCandidates([
      { agentId: "zone", zoneMatch: 1, typeMatch: 1, trustScore: 85, responseScore: 80, openOpportunities: 4, planWeight: 0 },
      { agentId: "premium", zoneMatch: 0, typeMatch: 1, trustScore: 95, responseScore: 95, openOpportunities: 1, planWeight: 1 },
    ]);
    expect(ranked[0].agentId).toBe("zone");
  });
});
```

- [ ] **Step 3: Implement scoring formula**

```ts
score =
  zoneMatch * 35 +
  typeMatch * 15 +
  trustScore * 0.20 +
  responseScore * 0.15 +
  max(0, 10 - openOpportunities) +
  planWeight * 5;
```

Exclude suspended agents, inactive subscriptions, unresolved conflicts, and agents at assignment capacity. Return reasons as structured labels.

- [ ] **Step 4: Implement public submission and admin assignment**

Public form includes contact, operation, type, zone, private address, expected price, description, photos, and communication consent. It does not publish a property.

Admin detail displays ranked candidates and lets admin select any eligible agent. Assignment creates audit `owner_submission.assigned`; reassignment logs previous and new agent.

- [ ] **Step 5: Run tests and schema reset**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/unit/agent-recommendation.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add supabase/migrations/0009_owner_submissions.sql src/features/owner-submissions src/app/'(public)'/tengo-una-propiedad src/app/'(admin)'/admin/propietarios tests/unit/agent-recommendation.test.ts src/types/database.ts
git commit -m "feat: add owner submissions and agent recommendations"
```

---

### Task 17: Verification Badges, Favorites, Saved Searches, and Internal Analytics

**Files:**
- Create: `supabase/migrations/0010_verification_analytics.sql`
- Create: `src/features/verifications/actions.ts`
- Create: `src/features/verifications/queries.ts`
- Create: `src/features/analytics/events.ts`
- Create: `src/features/analytics/queries.ts`
- Create: `src/features/properties/favorites-actions.ts`
- Create: `src/features/properties/saved-search-actions.ts`
- Create: `src/app/api/cron/saved-search-alerts/route.ts`
- Create: `src/app/(professional)/panel/page.tsx`
- Create: `src/app/(admin)/admin/page.tsx`
- Create: `tests/unit/verification-badges.test.ts`

**Interfaces:**
- Consumes: property, user, lead, visit, payment events.
- Produces: verification CRUD, `trackEvent`, dashboard metrics, favorites, saved searches.

- [ ] **Step 1: Add verification, favorite, saved-search, notification, and analytics tables**

Create `supabase/migrations/0010_verification_analytics.sql` with these tables:

- `property_verifications(property_id, kind, status, verified_by, verified_at, notes)` with kinds `identity`, `documentation`, `location`, `visited`, `photos`.
- `favorites(user_id, property_id, created_at)` unique by user/property.
- `saved_searches(user_id, name, filters jsonb, active, created_at)`.
- `notifications(user_id, type, payload jsonb, read_at, created_at)`.
- `analytics_events(id, event_name, user_id nullable, anonymous_session_id nullable, property_id nullable, agent_id nullable, source, metadata jsonb, created_at)` partition-ready by month.

Enable RLS on all five tables: users manage their own favorites and saved searches, public users cannot read raw analytics, agents read only aggregated metrics for their records, and admins manage verifications and read platform analytics.

- [ ] **Step 2: Write failing badge presentation tests**

```ts
import { describe, expect, it } from "vitest";
import { visibleVerificationBadges } from "@/features/verifications/presentation";

describe("verification badges", () => {
  it("returns only approved badges in fixed order", () => {
    expect(visibleVerificationBadges([
      { kind: "photos", status: "approved" },
      { kind: "identity", status: "approved" },
      { kind: "location", status: "pending" },
    ])).toEqual(["identity", "photos"]);
  });
});
```

- [ ] **Step 3: Implement verification actions**

Only admins can approve or revoke badges. Each change audits action, invalidates property cache, and recalculates `properties.verification_level` as the count of approved badge kinds. Public responses include badge kind only, never document paths or reviewer notes.

- [ ] **Step 4: Implement favorites and saved searches**

Authenticated users can toggle favorites and store sanitized `PropertySearchInput` JSON. Add `last_notified_at` to `saved_searches`. A daily secured route `/api/cron/saved-search-alerts` finds newly published matching properties since that timestamp, sends one consolidated email per saved search, and advances the timestamp only after successful delivery. Comparison list persists up to four property IDs in both account storage and `localStorage`; logged-out users use only local storage.

- [ ] **Step 5: Implement minimal analytics**

Track: `search_performed`, `property_viewed`, `whatsapp_clicked`, `visit_requested`, `lead_status_changed`, `property_confirmed`, `payment_submitted`, `payment_approved`. Do not store raw private addresses or verification documents in metadata. `property_viewed` increments `properties.view_count` through a rate-limited database function so popular sorting does not depend on scanning raw events.

Professional dashboard metrics: active properties, pending review, confirmation due, new leads, upcoming visits, total views, WhatsApp clicks. Admin dashboard metrics: active inventory by zone/type, searches, leads, visits, conversion, expired/paused, approved payments, promotion revenue.

- [ ] **Step 6: Run tests and schema reset**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/unit/verification-badges.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add supabase/migrations/0010_verification_analytics.sql src/features/verifications src/features/analytics src/features/properties src/app/'(professional)'/panel/page.tsx src/app/'(admin)'/admin/page.tsx tests/unit/verification-badges.test.ts src/types/database.ts
git commit -m "feat: add verification trust signals and analytics"
```
