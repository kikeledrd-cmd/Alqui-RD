# Ola 5 — Hardening and Launch

### Task 18: PWA, Accessibility, Abuse Protection, and Failure Recovery

**Files:**
- Create: `public/manifest.webmanifest`
- Create: `public/offline.html`
- Create: `src/app/manifest.ts`
- Create: `src/app/error.tsx`
- Create: `src/lib/rate-limit.ts`
- Create: `src/features/notifications/email-provider.ts`
- Modify: public forms and API routes.
- Create: `tests/unit/rate-limit.test.ts`

**Interfaces:**
- Consumes: app routes and public mutation endpoints.
- Produces: installable PWA metadata, offline fallback, rate limit API, notification adapter.

- [ ] **Step 1: Add PWA manifest and install metadata**

Manifest values:

```json
{
  "name": "Alqui-RD",
  "short_name": "Alqui-RD",
  "description": "Encuentra alquileres y propiedades en venta en el Gran Santo Domingo.",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0b1f3a",
  "lang": "es-DO",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

Do not implement push notifications in this release.

- [ ] **Step 2: Write failing rate-limit test**

```ts
import { describe, expect, it } from "vitest";
import { SlidingWindowLimiter } from "@/lib/rate-limit";

describe("SlidingWindowLimiter", () => {
  it("blocks the fourth request in a three-request window", () => {
    const limiter = new SlidingWindowLimiter(3, 60_000);
    expect(limiter.check("ip", 0).allowed).toBe(true);
    expect(limiter.check("ip", 1).allowed).toBe(true);
    expect(limiter.check("ip", 2).allowed).toBe(true);
    expect(limiter.check("ip", 3).allowed).toBe(false);
  });
});
```

- [ ] **Step 3: Apply abuse controls**

Apply limits:

- Login: 5 attempts per 15 minutes per IP/email hash.
- Agent registration: 3 per hour per IP.
- WhatsApp lead endpoint: 20 per hour per anonymous session and 60 per hour per IP.
- Visit requests: 5 per day per phone/property pair.
- Owner submissions: 3 per day per phone.
- Media signed URLs: 120 per hour per agent.

Use a provider interface so in-memory limiter is used only in tests; production adapter must use a shared Redis-compatible store.

- [ ] **Step 4: Add accessible and recoverable error UI**

Requirements:

- Every form field has a label, described error, and keyboard focus on first invalid field.
- Carousel supports keyboard and has visible focus.
- Dialogs trap focus.
- Color is never the only state signal.
- `src/app/error.tsx` offers retry and home actions.
- Public forms retain values after network errors.
- Search cards use progressive images with fixed aspect ratio to prevent layout shift.

- [ ] **Step 5: Add notification provider interface**

```ts
export interface NotificationProvider {
  send(input: { to: string; subject: string; text: string; html: string }): Promise<{ messageId: string }>;
}
```

Provide `ConsoleNotificationProvider` for local development and a Resend-compatible implementation selected by environment. Notification failures must be logged but must not roll back the domain transaction.

- [ ] **Step 6: Run tests and Lighthouse smoke**

Run:

```bash
pnpm test tests/unit/rate-limit.test.ts
pnpm typecheck
pnpm build
pnpm dev
```

Then run Lighthouse mobile against `/`, `/buscar`, and one property page. Acceptance floors: Performance 75, Accessibility 90, Best Practices 90, SEO 90.

- [ ] **Step 7: Commit**

```bash
git add public src/app/manifest.ts src/app/error.tsx src/lib/rate-limit.ts src/features/notifications tests/unit/rate-limit.test.ts
git commit -m "feat: harden PWA accessibility and abuse protection"
```

---

### Task 19: End-to-End Launch Path and Deployment Readiness

**Files:**
- Create: `tests/e2e/agent-publication.spec.ts`
- Create: `tests/e2e/public-search-lead.spec.ts`
- Create: `tests/e2e/visit-flow.spec.ts`
- Create: `tests/e2e/payment-approval.spec.ts`
- Create: `scripts/seed-e2e.ts`
- Create: `docs/operations/deployment.md`
- Create: `docs/operations/admin-runbook.md`
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: complete MVP.
- Produces: reproducible seed, CI gate, launch runbook, and verified release criterion.

- [ ] **Step 1: Create deterministic E2E seed**

Seed users:

```text
admin@alqui-rd.test / AlquiRD-Test-Admin-2026!
agent.pending@alqui-rd.test / AlquiRD-Test-Pending-2026!
agent.approved@alqui-rd.test / AlquiRD-Test-Agent-2026!
visitor@alqui-rd.test / AlquiRD-Test-Visitor-2026!
```

Seed one approved agent with active trial, one pending agent, one draft property, one published property, and one submitted payment proof fixture.

- [ ] **Step 2: Write agent publication E2E test**

Test:

1. Pending agent cannot submit property.
2. Admin approves pending agent.
3. Agent creates draft, uploads three images, selects cover, submits.
4. Admin approves property.
5. Property becomes publicly accessible by code.

- [ ] **Step 3: Write public search and lead E2E test**

Test:

1. Search rent + Santo Domingo Este + apartment.
2. Open property.
3. Verify carousel and public location.
4. Click WhatsApp.
5. Intercept redirect and confirm lead exists with source and property.

- [ ] **Step 4: Write visit and payment E2E tests**

Visit: anonymous visitor requests date/time block; agent confirms; marks completed; lead becomes `visit_completed`.

Payment: agent uploads proof; plan remains unchanged; admin approves; subscription end extends exactly 30 days; audit log exists.

- [ ] **Step 5: Add CI workflow**

`.github/workflows/ci.yml` must run on pull requests and main pushes:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: pnpm/action-setup@v4
    with: { version: 10 }
  - uses: actions/setup-node@v4
    with: { node-version: 22, cache: pnpm }
  - run: pnpm install --frozen-lockfile
  - run: pnpm supabase start
  - run: pnpm supabase db reset
  - run: pnpm verify
  - run: pnpm playwright install --with-deps chromium
  - run: pnpm test:e2e
```

- [ ] **Step 6: Write deployment and admin runbooks**

`deployment.md` must include environment variables, Supabase migration command, storage buckets, cron schedule `0 13 * * *` UTC, which runs at 9:00 a.m. in America/Santo_Domingo, custom domain, backups, rollback, and smoke checks.

`admin-runbook.md` must include agent approval, moderation reasons, payment proof review, owner assignment, property pause/reactivation, incident escalation, and private data handling.

- [ ] **Step 7: Run full verification**

Run:

```bash
pnpm supabase db reset
pnpm verify
pnpm test:e2e
```

Expected:

- Unit and integration tests PASS.
- Production build PASS.
- All four E2E suites PASS.
- No public API response contains `private_address`, verification document paths, or payment proof paths.

- [ ] **Step 8: Commit**

```bash
git add tests scripts docs/operations .github/workflows/ci.yml
git commit -m "test: verify Alqui-RD launch journey"
```

---

## Implementation Waves

Execute tasks in this dependency order:

1. **Foundation:** Tasks 1–4.
2. **Supply engine:** Tasks 5–10.
3. **Conversion engine:** Tasks 11–13.
4. **Business engine:** Tasks 14–17.
5. **Hardening and launch:** Tasks 18–19.

A wave is accepted only when every task in it passes its listed commands and its commits are reviewable independently.

## Spec Coverage Review

- Public search, filters, home, cards, carousel, property detail: Tasks 5 and 10.
- Optional visitor registration, favorites, saved searches, comparison: Tasks 3 and 17.
- Agent registration, approval, publishing lock, mini office: Tasks 3, 6, 7, 10.
- Property form, media, privacy, moderation, trust: Tasks 7–10.
- WhatsApp lead traceability and commercial states: Task 11.
- Hybrid visit scheduling: Task 12.
- 15/30-day renewal and automatic pause: Task 13.
- Manual payments, subscriptions, promotions, future provider abstraction: Task 14.
- Agency teams and roles: Task 15.
- Private owner intake and ranked assignment: Task 16.
- Verification levels and analytics: Task 17.
- PWA, mobile, accessibility, security, failure recovery: Task 18.
- End-to-end launch criterion, CI, deployment, operations: Task 19.

No native app, automatic card payment, AI pricing, digital contract, internal chat, bank integration, 3D tour, legal contract workflow, or national expansion is included.
