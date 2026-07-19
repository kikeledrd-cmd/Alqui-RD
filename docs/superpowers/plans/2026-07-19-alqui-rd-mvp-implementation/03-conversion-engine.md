# Ola 3 — Conversion Engine

### Task 11: WhatsApp Lead Capture and Commercial Timeline

**Files:**
- Create: `supabase/migrations/0006_leads_visits.sql`
- Create: `src/features/leads/types.ts`
- Create: `src/features/leads/service.ts`
- Create: `src/features/leads/actions.ts`
- Create: `src/app/api/leads/whatsapp/route.ts`
- Create: `src/components/property/whatsapp-contact-button.tsx`
- Create: `src/app/(professional)/panel/contactos/page.tsx`
- Create: `src/app/(professional)/panel/contactos/[id]/page.tsx`
- Create: `tests/unit/whatsapp-link.test.ts`

**Interfaces:**
- Consumes: public property code and responsible agent.
- Produces: `createLead`, `recordLeadActivity`, `buildWhatsAppUrl`, and POST `/api/leads/whatsapp`.

- [ ] **Step 1: Create lead and visit tables**

In `0006_leads_visits.sql` create `leads`, `lead_activities`, and `visit_requests`. Required lead columns: property, agent, agency nullable, registered user nullable, visitor name/phone/email nullable, source, medium, status, commission applicable flag, created/updated timestamps. Use a database trigger to append a `lead_activities` row whenever status changes. Enable RLS: agents read their assigned leads and visits, agency supervisors/admins read their agency records, admins read all, and anonymous users can only insert through security-definer server functions.

- [ ] **Step 2: Write failing WhatsApp URL tests**

```ts
import { describe, expect, it } from "vitest";
import { buildWhatsAppUrl } from "@/features/leads/whatsapp";

describe("buildWhatsAppUrl", () => {
  it("embeds the property code and public URL", () => {
    const url = buildWhatsAppUrl({ phone: "18095551234", code: "ARD-000123", propertyUrl: "https://alqui-rd.com/inmueble/ARD-000123" });
    expect(url).toContain("wa.me/18095551234");
    expect(decodeURIComponent(url)).toContain("ARD-000123");
  });
});
```

- [ ] **Step 3: Implement lead creation with idempotency**

`POST /api/leads/whatsapp` accepts:

```ts
{
  propertyCode: string;
  source: "google" | "instagram" | "facebook" | "direct" | "other";
  anonymousSessionId: string;
}
```

Use idempotency key SHA-256 of `propertyId + anonymousSessionId + current UTC day`. Repeated clicks the same day update `last_contact_at` rather than creating duplicate leads. Return `{ leadId, whatsappUrl }`.

If WhatsApp URL generation fails, the lead insert remains committed and the response returns HTTP 200 with a fallback `tel:` URL.

- [ ] **Step 4: Implement professional lead pipeline actions**

```ts
export async function changeLeadStatus(input: { leadId: string; status: LeadStatus; note?: string }): Promise<ActionResult<{ status: LeadStatus }>>;
export async function addLeadNote(input: { leadId: string; note: string }): Promise<ActionResult<{ activityId: string }>>;
```

Allowed transitions must prevent `closed` from returning to `new`. Agency supervisors/admins can reassign team leads; independent agents cannot reassign outside themselves.

- [ ] **Step 5: Build contacts list and timeline**

List columns: created date, property code/title, visitor, channel, current status, last activity. Detail renders all activities chronologically and provides status buttons.

- [ ] **Step 6: Run tests, reset schema, and build**

Run:

```bash
pnpm supabase db reset
pnpm supabase gen types typescript --local > src/types/database.ts
pnpm test tests/unit/whatsapp-link.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add supabase/migrations/0006_leads_visits.sql src/features/leads src/app/api/leads src/components/property/whatsapp-contact-button.tsx src/app/'(professional)'/panel/contactos tests/unit/whatsapp-link.test.ts src/types/database.ts
git commit -m "feat: capture WhatsApp leads and commercial activity"
```

---

### Task 12: Hybrid Visit Scheduling

**Files:**
- Create: `src/features/visits/schema.ts`
- Create: `src/features/visits/actions.ts`
- Create: `src/features/visits/state-machine.ts`
- Create: `src/components/property/visit-request-form.tsx`
- Create: `src/app/(professional)/panel/visitas/page.tsx`
- Create: `tests/unit/visit-state-machine.test.ts`

**Interfaces:**
- Consumes: existing lead or creates lead from visitor details.
- Produces: `requestVisit`, `respondToVisit`, `completeVisit`, `canTransitionVisit`.

- [ ] **Step 1: Write failing visit state tests**

```ts
import { describe, expect, it } from "vitest";
import { canTransitionVisit } from "@/features/visits/state-machine";

describe("visit state machine", () => {
  it("allows requested to confirmed", () => {
    expect(canTransitionVisit("requested", "confirmed")).toBe(true);
  });

  it("blocks completed to requested", () => {
    expect(canTransitionVisit("completed", "requested")).toBe(false);
  });
});
```

- [ ] **Step 2: Implement visit schema and states**

States: `requested`, `confirmed`, `reschedule_proposed`, `cancelled`, `completed`, `no_show`. Request fields: preferred date, one of `morning|afternoon|evening`, visitor name, phone, email optional, consent boolean.

- [ ] **Step 3: Implement request and agent response actions**

`requestVisit` must create or reuse a lead, insert visit, set lead status `visit_requested`, and notify agent. `respondToVisit` can confirm, propose an exact ISO datetime, or cancel with reason. `completeVisit` records outcome `attended_interested|attended_not_interested|no_show|negotiation|closed` and synchronizes lead status.

- [ ] **Step 4: Build visitor form and professional agenda**

Visitor form must work without login. Agenda groups upcoming visits by date, exposes confirm/reschedule/cancel actions, and shows contact/property context.

- [ ] **Step 5: Run tests and build**

Run:

```bash
pnpm test tests/unit/visit-state-machine.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/features/visits src/components/property/visit-request-form.tsx src/app/'(professional)'/panel/visitas tests/unit/visit-state-machine.test.ts
git commit -m "feat: add hybrid property visit scheduling"
```

---

### Task 13: Inventory Renewal and Automatic Pausing

**Files:**
- Create: `src/features/renewals/policy.ts`
- Create: `src/features/renewals/service.ts`
- Create: `src/app/api/cron/property-expiration/route.ts`
- Create: `tests/unit/renewal-policy.test.ts`

**Interfaces:**
- Consumes: property operation, status, confirmation dates.
- Produces: `nextConfirmationDate`, `processPropertyRenewals(now)`, cron endpoint.

- [ ] **Step 1: Write failing renewal policy tests**

```ts
import { describe, expect, it } from "vitest";
import { nextConfirmationDate } from "@/features/renewals/policy";

describe("nextConfirmationDate", () => {
  it("uses 15 days for rentals", () => {
    expect(nextConfirmationDate("rent", new Date("2026-07-01T12:00:00Z")).toISOString()).toBe("2026-07-16T12:00:00.000Z");
  });

  it("uses 30 days for sales", () => {
    expect(nextConfirmationDate("sale", new Date("2026-07-01T12:00:00Z")).toISOString()).toBe("2026-07-31T12:00:00.000Z");
  });
});
```

- [ ] **Step 2: Implement policy**

`nextConfirmationDate(operation, from)` adds exactly 15 or 30 calendar days in UTC. `graceDeadline(due)` adds 3 days.

- [ ] **Step 3: Implement daily renewal processor**

At each run:

1. Published properties due now become `confirmation_due`.
2. `confirmation_due` properties older than 3 days become `paused`.
3. Create one notification per transition.
4. Create one audit log per transition.
5. Return counts `{ due, paused, notifications }`.

Use batched SQL/RPC to avoid per-row network calls.

- [ ] **Step 4: Secure the cron route**

`GET /api/cron/property-expiration` requires `Authorization: Bearer <CRON_SECRET>`. Return 401 on mismatch and JSON counts on success.

- [ ] **Step 5: Add agent confirmation action**

`confirmPropertyAvailability(propertyId)` changes `confirmation_due` or `paused` to `published`, updates `confirmed_at`, and recalculates `next_confirmation_at`. If an admin moderation hold exists, return `moderation_hold`.

- [ ] **Step 6: Run tests and build**

Run:

```bash
pnpm test tests/unit/renewal-policy.test.ts
pnpm typecheck
pnpm build
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/features/renewals src/app/api/cron tests/unit/renewal-policy.test.ts
git commit -m "feat: automate property availability renewal"
```
