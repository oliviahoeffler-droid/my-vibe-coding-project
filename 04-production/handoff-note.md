# Engineering Handoff Note

> Module 4 · Production Specs. Open the black box, make the build legible to an engineer.

## What this is

_One paragraph an engineer can read in 60 seconds._

The 60-second read
FirstCall is a multi-tenant web app that responds to a new contractor lead within seconds: when a homeowner submits a website form, the app sends an SMS within ~10 seconds, follows with an AI voice call, and tries to book an appointment on a sales rep's calendar, then sends confirmations to the lead and the rep. It's a TanStack Start (React 19) app on Lovable Cloud (Postgres with row-level security). All outreach logic lives in one server-side "engine" module so the web form, SMS, and voice paths share a single booking flow. Everything external is currently simulated in test mode: texts, calls, emails, and calendar connections are logged to a per-lead timeline instead of going out over real providers. The UI, data model, scheduling rules, AI texting agent, booking page, stats, and recovery queue are real and working; the provider integrations (Twilio, ElevenLabs, Google Calendar, email delivery) are the remaining work.

## Architecture (plain language)

- **Frontend:** Stack: TanStack Start v1 (React 19, Vite 7), Tailwind CSS v4, shadcn-style components. TypeScript throughout. Organization: code is grouped by feature under src/features/<feature>/ — workspace, leads, appointments, settings, speed-to-lead, recovery, booking, auth, landing. Each screen component is named <ScreenName>Screen after its PRD screen. Within a feature: *.data.ts holds queries/mutations, *.logic.ts holds pure rules, components only render. Routes: src/routes/ files are thin — route config plus a screen import. Key routes: / (landing), /auth, /app (dashboard shell with leads list + slide-in detail panel), /app/leads/$id, /app/appointments, /app/settings (General + Rep calendars tabs), /app/stats, /app/recovery, /book/$token (public booking page). Data fetching: route loaders call context.queryClient.ensureQueryData(...); components use useSuspenseQuery. Realtime updates on the leads list and lead timeline come from the database's realtime channel.
- **Backend / data:** Platform: Lovable Cloud — Postgres with row-level security (RLS), auth (email/password + Google + Lovable sign-in), and server functions via createServerFn from @tanstack/react-start. No separate API server. Multi-tenancy: every app table carries company_id. RLS policies use an is_company_member security-definer function. Roles live in company_members (owner / member). Tables: companies (intake key/secret, timezone, business hours, appointment length, test-mode flag), company_members, services, reps, leads (status, assigned rep, booking token, first_text_at, call_started_at), lead_events (the timeline: sms_out, sms_in, call, email, booked, calendar, each with body + meta), appointments. The engine: src/lib/server/engine.server.ts owns all outreach, booking, and confirmation logic. Routes and server functions only validate input and call the engine, so the web form, SMS, and voice flows share one booking path. Pure rules: src/lib/slots.ts computes open time slots (business hours, appointment length, existing appointments) and does round-robin rep assignment. Covered by tests in src/test/slots.test.ts (plus stats-rule tests locking in the 10-second target and booking-rate definition). AI texting agent: the engine runs an agent loop (AI SDK, OpenAI Responses API) with two tools — get_available_slots and book_appointment — so a lead's text reply can turn into a booked slot without a human. Public endpoints: the lead intake webhook (/api/public/leads/$key) and the booking page (/book/$token) bypass login, so they use the privileged server client only after verifying the per-company intake secret or the per-lead booking token. The webhook also has a honeypot field and CORS handling, and returns HTML or JSON depending on the caller.
- **Key flows:** Lead intake: contractor's website form POSTs to the webhook (snippet provided in Settings) → lead row created, rep assigned round-robin → outreach starts.
Outreach: engine sends the first SMS (target: under 10 seconds), then starts the AI voice call. In test mode both are logged as lead_events with meta.mode = "test".
Booking: three paths into the same bookAppointment engine function — the AI texting agent's tool call, the voice call, or the lead picking a slot on the public booking page (token-verified).
Confirmation: on booking, the engine writes confirmation text/email events for the lead and the assigned rep, and creates the appointment row.
Recovery: leads stuck at no_answer appear in the Recovery queue; a retry action re-runs outreach through the engine (also test-mode).
Stats: the Speed-to-lead screen computes average seconds-to-first-text, booking rate, and leads-by-status from leads + lead_events.

## What's solid vs. what's duct tape

| Area | State | Notes |
|---|---|---|
| Data model and tenancy | solid | company_id everywhere, RLS enforced, membership helper functions, creator-becomes-owner trigger. Tested implicitly by every screen. |
| All providers are simulated. | rough | SMS, voice, and email "providers" are engine functions that write lead_events rows. Swapping in Twilio/ElevenLabs/a mail provider is designed to be a one-place change, but that swap is unwritten and untested. |

## Risks & assumptions for the team

Provider integration risk (high): the test-mode seams are clean but unproven. Twilio delivery webhooks, ElevenLabs call control, and email sending each need signature verification and idempotency handling that doesn't exist yet.
10-second promise depends on provider latency: the app timestamps first_text_at, but real-world SMS send time includes queue + carrier delay. The stat measures our response, not delivery.
Assumption: one timezone per company. Slots are computed in the company's timezone; reps and leads are assumed to share it. Multi-timezone ops would need per-rep/per-lead timezones.
Assumption: round-robin is good enough. No capacity weighting, no rep-level hours, no conflict checking against a real calendar (because calendars aren't connected).
Assumption: the AI texting agent stays within its tools. It can only read slots and book; prompt-injection via lead replies is mitigated by the narrow tool set, but tool-call guardrails should be re-reviewed before real SMS.
Realtime at scale: the dashboard subscribes to leads/events per company; fine for demo volume, untested under bursty intake.
Secrets: the intake secret is a shared bearer-style secret per company; rotation UI doesn't exist.

## How to run it

```
Prerequisites: Bun (or Node 18+), and the project's environment variables (provided automatically in the Lovable environment; for local runs you need the Cloud project's URL and publishable key in .env).
Try the core loop end to end:

Sign up at /auth — you're made owner of a new company workspace.
In Settings → General, copy the webhook URL + form snippet; note the intake secret. Add a service and a rep.
Send a test lead (dashboard button, or POST the webhook with the secret).
Watch the lead appear on the dashboard; open the side panel to see the timeline — first text, call, and booking events appear (test mode).
Use "Reply as the lead" to text back a time preference — the AI agent offers open slots and books one.
Check Appointments, Speed-to-lead (stats), Settings → Rep calendars (rep day view), and Recovery (retry queue).
The public booking page at /book/<token> works without login, using the lead's token.
Where to start as the incoming engineer: read AGENTS.md (architecture rules), then src/lib/server/engine.server.ts (the heart), then src/lib/slots.ts + its tests. The first real task is almost certainly one of the three blocked items: Google Calendar OAuth, Twilio/ElevenLabs, or a verified email sender domain.
```
