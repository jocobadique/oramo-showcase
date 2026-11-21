# Architecture

A deeper look at how Oramo is built. This document is for engineers evaluating the depth of the project.

## Mental model: one database, many clinics

Oramo is a **shared-database, row-isolated** multi-tenant SaaS. A **clinic** is the tenant, and everything else hangs off it:

```
Clinic (tenant) ─┬─ Branches ─── Branch staff (dentists / staff)
                 ├─ Services (branch-scoped or clinic-wide)
                 ├─ Patients ─┬─ Dental chart ─── Tooth records (append-only)
                 │            ├─ Medical history (structured jsonb)
                 │            ├─ Charges ─── Payments
                 │            └─ Files (private storage)
                 ├─ Appointments ─── Notifications log
                 └─ Subscription ─── Invoices · SMS credit ledger
Profiles map auth.users → one clinic + a role (owner | dentist | staff | patient)
```

The guiding rule is that **app code never filters by `clinic_id`. The database does.** Every query is written as if the table held only the caller's clinic.

## Security: RLS is the contract

- **Helper functions resolve the caller's tenant.** `auth_clinic_id()`, `auth_role()` and `auth_patient_ids()` are `SECURITY DEFINER` functions that read the caller's `profiles` row. They bypass RLS so that policies don't recurse.
- **Every tenant table has a policy of the form `using (clinic_id = auth_clinic_id())`**, with role-aware variations. For example, only an owner can update the clinic record.
- **Privilege escalation is closed with column grants.** Authenticated users may update only `full_name` on their own profile. `role`, `clinic_id` and `is_active` change only through owner-checked definer RPCs: `create_clinic_with_branch`, `accept_invite`, `set_staff_role` and `set_staff_active`.
- **Storage follows the same rule.** Patient files sit in a **private** bucket under `<clinic_id>/<patient_id>/<file>`. Storage policies require the first path segment to equal `auth_clinic_id()`, and files are served only through short-lived signed URLs.
- **Isolation is tested.** A SQL test seeds two clinics, impersonates a Clinic A user, and asserts that every read and write against Clinic B fails. It runs inside a transaction that rolls back.
- **Service-role keys are server-only.** They're used by the booking route, the billing webhook and the Edge Function, and never reach a client bundle.

### Pre-deploy security audit

Before the first deploy, a dedicated audit migration closed three issues:

1. **Clinic takeover via signup metadata (critical).** The new-user trigger copied `clinic_id` and `role` from `raw_user_meta_data`, which the caller controls. Combined with a public RPC that returned clinic IDs, anyone could have signed up as the owner of any clinic. Now the trigger trusts `full_name` only, and membership is granted solely by the definer RPCs.
2. **Public booking RPC callable directly.** The anon key could call the booking RPC through PostgREST and skip every defense in the route handler. Each booking fires a metered SMS, so this was also a cost risk. The RPC is now `service_role`-only, and the route calls it after its checks pass.
3. **Internal functions reachable by API roles.** Supabase's default privileges grant `EXECUTE` directly to `anon` and `authenticated`, so the earlier `revoke … from public` lines didn't actually remove that access. The audit revoked it explicitly on the notification and billing internals.

## The public booking path

`/<slug>` is public, sits outside the dashboard, and is not gated by the proxy. Static routes take precedence over the dynamic segment. The page is deliberately narrow:

```
 visitor ──► GET /<slug> ──► get_public_clinic_by_slug()   (public-safe fields only:
                                                            name, branding, active
                                                            branches + services)

 visitor ──► POST /api/public/book
               1. honeypot          (bots silently dropped)
               2. CAPTCHA hook      (Cloudflare Turnstile when configured)
               3. server validation (PH mobile format, future time, required fields)
               4. rate limit        (5 / 10 min per IP · 4 / hour per mobile)
               5. create_public_appointment()  ── service role, forces status = 'requested'
```

- **Reads never touch tenant tables.** The page's TypeScript types are kept separate from the dashboard's row types, so patient, appointment and staff shapes can't leak into it.
- **Rate limiting needs no new dependencies.** It uses Upstash's REST API (`INCR` + `EXPIRE`) in production and falls back to an in-memory window in development. It fails open, so a limiter outage never blocks a real patient.
- **Availability** is generated from the branch's operating hours and the service's duration. These are *requests* that staff confirm, so the page never reads existing appointments to compute free and busy times.

## Notifications: fired by the database, sent exactly once

```
 appointments INSERT/UPDATE ─► trigger ─► pg_net ─┐
                                                   ├─► Edge Function `notify` ─► SmsProvider  (Semaphore)
 pg_cron */15 min ─► reminder sweep ─► pg_net ────┘                          └► EmailProvider (Resend)
                                                              │
                                                              ▼
                                                notifications log (also the SMS meter)
```

- **The app's write paths contain no notification code.** Changing an appointment's status is enough, and the database fires the event.
- **Delivery is exactly-once, guarded at three layers:**
  1. a unique index on `(appointment_id, type, channel)`;
  2. **claim-then-send**: the function inserts a `queued` row first, and a duplicate insert means the work is already handled;
  3. **bounded retry with optimistic concurrency** (`update … where status = <seen>`), so overlapping sweeps can't both send.
- **Providers are swappable.** `SmsProvider` and `EmailProvider` are interfaces with a factory. Adding a different SMS gateway takes one class and one `case`.
- **Clinics edit their own templates.** Placeholders like `{patient_name}` and `{date}` are validated, and the editor shows a live preview with the SMS segment count, because SMS is the metered cost.

## Dental charting

- **FDI two-digit notation** (the standard in PH dental schools). Permanent teeth are `11–48`, primary teeth `51–85`, and mixed dentition is supported.
- **Five-surface teeth.** Mesial, distal, buccal and lingual surfaces surround occlusal or incisal. One charting action writes one row per selected surface. Whole-tooth conditions (crown, implant, missing, …) are written as a single row.
- **History is append-only, and the chart is derived from it.** `tooth_records` are never overwritten. The current chart is a **pure function** of that history: the latest non-cancelled record per surface, whole-tooth conditions overriding, and planned work drawn as an overlay. Deleting a record is a separate, explicit correction step.
- **Status is never shown by color alone.** Every state pairs a color with an icon or pattern, in light and dark mode.

## AI assist: AI fills in, people save

Two features, one rule: the model only fills in a form, and nothing is written until a person confirms.

```
 🎤 MediaRecorder (≤60 s) ──► /api/ai/transcribe ──► OpenAI gpt-4o-transcribe ──► text (editable)
 text ──► /api/ai/quick-book ──► Claude Haiku 5.5 ──► validate ──► prefilled appointment form
 text ──► /api/ai/chart-notes ─► Claude Sonnet 5.5 ─► validate ──► tooth-record review list ──► save
```

- **Structured output in the app's own vocabulary.** Both prompts return JSON constrained by a Zod schema: service and dentist ids from lists the server sends, FDI tooth numbers, surfaces `M D B L O I`, and the controlled condition, treatment and status enums. Dates come from a three-week calendar in the request, because models are bad at date math.
- **The server trusts nothing it didn't send.** Unknown ids, impossible teeth, surfaces a tooth doesn't have, past or closed-day times are dropped and turned into "please check" hints. A treatment marked done implies its resulting condition, so the chart colors correctly.
- **Minimal data to the model.** Quick book sends the typed line plus service names, dentist names and branch hours. Patient matching (mobile first, then name) happens in Postgres under RLS, never in the model. Chart notes are sent without the patient's name or id. Audio goes to the speech-to-text provider only while recording and is not stored; no text or audio is logged.
- **Metered per plan.** `ai_consume()` checks and counts an assist atomically (row lock on the subscription) against the plan's monthly allowance: trial 50, Starter 200, Pro/Group 600 per billable branch. Failed or empty reads are refunded with `ai_refund()`. The `ai_usage` log has no table grants at all; only these functions touch it.
- **Graceful failure.** Rate limits per user and per clinic (tighter on the public demo), friendly messages for timeouts, safety refusals, and an empty provider balance or spend cap. Typing and charting by hand always keep working.

## Billing: two separate systems

| | **Clinical billing** | **SaaS billing** |
| --- | --- | --- |
| Who pays whom | patient → clinic | clinic → Oramo |
| Tables | `charges`, `payments` | `subscriptions`, `billing_invoices`, `sms_credit_ledger` |
| Derived in the DB | `paid_amount` and `status`, via triggers | SMS allowance and usage, via `sms_allowance_state()` |

### Subscription state machine

```
trialing (14 days, no card) ──► active ──► past_due ──(grace)──► read-only
        │                         │
        └── lapsed ─► read-only   └──► canceled / expired ─► read-only
```

- **One function, `computeEntitlements()`, decides what a clinic can do.** The client hook and the server guard both call it, so gating can't drift between them.
- **Data is never hard-locked.** An expired clinic drops to *read-only* access. Dental records have legal retention requirements, so lapsing must never destroy them.
- **Pricing is per active branch.** A `BEFORE INSERT` trigger enforces the branch cap. Changes to the branch count take effect at the next billing cycle, with no mid-cycle proration.
- **PayMongo webhooks are the source of truth.** The handler verifies the HMAC-SHA256 signature on the **raw** body with a timing-safe comparison, deduplicates by provider event ID, and writes with the service role. Entitlements are never granted from the checkout redirect; the UI polls until the webhook lands.
- **SMS metering is idempotent.** A trigger on the notifications log decrements credits for segments beyond the plan's allowance. A unique partial index makes sure a re-run can't charge twice.

## Routing

| Route | Auth | Purpose |
| --- | --- | --- |
| `/` | public | Marketing landing page |
| `/login`, `/signup`, `/forgot-password`, `/reset-password` | public | Auth; `/auth/confirm` verifies email OTPs server-side |
| `/onboarding` | signed in, no clinic | Two-step wizard: clinic, then first branch (atomic RPC) |
| `/accept-invite` | signed in, pending invite | Staff join an existing clinic |
| `/dashboard/*` | **required** | Dashboard (today) · Appointments · Patients · Branches · Staff* · Notifications* · Billing* · Settings* |
| `/<slug>` | **public** | Clinic-branded booking page |
| `/api/public/book` | public | Hardened booking write |
| `/api/billing/*` | owner / signed webhook | Subscribe, top up, cancel, webhook |

\* owner-only. These are hidden in the navigation **and** guarded server-side in the route layout.

`proxy.ts` (Next.js 16's renamed middleware) refreshes the session and guards the dashboard, onboarding and invite routes. The dashboard layout is the authoritative check, routing users by state: signed out, no clinic, pending invite, or a role.

## State management

| Kind | Mechanism | Notes |
| --- | --- | --- |
| Server state | TanStack Query hooks → typed data-access layer (`lib/db/*`) | Components never call Supabase directly. Mutations are optimistic with rollback. |
| Session | TanStack Query (`useSession`) | Logging in or out re-scopes every cached query. |
| Client UI state | Zustand | Active branch, calendar view, open sheets, sidebar mode. Never server data. |

## Built for the Philippines

- **Asia/Manila time.** Everything is stored as UTC `timestamptz` and displayed in Manila time. PH has a fixed UTC+8 offset with no DST, so conversion is exact, and it all goes through one module.
- **PH mobile validation**, peso pricing, and a local SMS gateway.
- **PDA medical-history questionnaire** stored as typed, structured data. Stored JSON is treated as untrusted and merged onto a complete default.
- **PayMongo**, with GCash, Maya and cards, which is how PH clinics actually pay.

## UX conventions

- **A skeleton for every data-driven view, never a spinner.** Each skeleton matches its layout so content swaps in without layout shift. The calendar skeleton reuses the real grid's geometry.
- **A full or mini sidebar**, with hover and focus popovers in mini mode, persisted per user. Phones and tablets get a full-label drawer instead.
- **Branded controls throughout.** There are no OS-native `<select>` or date widgets; `Select`, `DatePicker` and `TimePicker` are built on one popover primitive.
- **Toasts for transient results; inline alerts for messages that belong in place.**

## Migration history

| Migration | Concern |
| --- | --- |
| `0001_schema` | Enums, tables, indexes, new-user trigger |
| `0002_rls` | RLS helpers, policies, explicit role grants |
| `0003_public_booking` | Public-safe booking RPCs |
| `0004_auth_onboarding` | Invites, onboarding RPCs, column-level lockdown |
| `0005_notifications` | Notifications log, per-clinic settings, triggers, `pg_cron` sweep |
| `0006_patient_records_and_settings` | Medical history, clinical billing, patient files, staff RPCs |
| `0007_saas_billing` | Subscriptions, plan limits, invoices, SMS credits, webhook dedupe |
| `0008_harden_charge_trigger_search_path` | Pinned `search_path` (Supabase advisor fix) |
| `0009_security_hardening` | The pre-deploy audit fixes described above |
| `0010`–`0016` | Function privileges, pricing and founding clinics, legal consent, demo requests, plan ladder and team caps, audit log, clinic reports |
| `0017`–`0021` | Two-step verification enforced in RLS, platform admin portal, setup checklist, reserved slugs, SMS credit adjustments |
| `0022`–`0028` | Multi-service visits, atomic SMS top-ups, chairs per branch, patient change/cancel links, HMO claims, clinic two-step rule |
| `0029_service_follow_up` | Per-service follow-up intervals (recalls) |
| `0030_rls_initplan` | Policy helpers evaluated once per query: `(select auth_clinic_id())` |
| `0031_ai_allowance` | Monthly AI allowance per plan, atomic consume and refund |

## Stack summary

Next.js 16 · React 19 · TypeScript (strict) · Tailwind CSS v4 · shadcn/ui · Supabase (Postgres + RLS + Auth + Storage + Edge Functions + pg_cron) · TanStack Query v5 · Zustand · Claude (Anthropic SDK) · OpenAI speech-to-text · Zod · PayMongo · Semaphore · Resend · Upstash · deployed on Vercel.
