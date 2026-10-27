# Full-Stack: Data, Access Rules, Edge Cases, Deploy

> Module 5 · Full-Stack. Add data schemas, access rules, and edge cases; stress-test and deploy.

## Deployed link

_The working, shareable link that survives real users._

https://instant-lead-booking-buddy.lovable.app

## Data schema

| Entity | Key fields | Notes |
|---|---|---|
| Company timezone | America/Chicago, hours 08:00–17:00, duration 60 minutes, test mode true | Real editable configuration. Validate timezone and coherent hours/duration on both server and database boundaries |
| Rep day boundaries and status-chart days use browser-local dates | settings.data.ts, RepCalendarsScreen.tsx, speedToLead.logic.ts | Inconsistent with company-timezone appointment formatting. Compute query bounds and display days in company timezone |

## Access rules

_Who can see / do what? Where are the auth boundaries?_

Verified present state
All seven live tables have RLS enabled.
Companies: members/creator can read; creator can insert; owners can update/delete.
Memberships: members can read; owners have broad management access.
Leads, events, appointments, reps and services: company members currently have ALL operations, including delete.
Effective privilege checks show anonymous SELECT privileges on all seven tables, but the inspected policies are authenticated-only. A table grant is not an RLS policy: this is not evidence that anonymous users can currently read rows. Revoke unnecessary grants and preserve RLS; do not treat grants alone as proof of a leak.
No two-company adversarial access test was performed. Prior handoff claims that tenancy is “solid” are stronger than this audit can establish.
Target rule matrix
Resource	Member	Owner	Anonymous / privileged server
Safe company settings	Read own company	Read/update own; create with created_by = auth.uid()	No anonymous table access
Memberships	Read own company; cannot promote self	Manage own company via validated operation; cannot remove/demote last owner	Creator trigger creates initial owner; roles never from client storage
Services / reps	Read own	Manage own	No anonymous CRUD; do not add rep-only access until a verified user↔rep mapping exists
Leads	Read own; permitted operational edits	Same, with explicit deletion/retention policy	Public intake validates credential/public-form controls before narrow creation; cannot choose arbitrary tenant
Appointments	Read own; booking through validated operation	Same; lifecycle changes only when in scope	Token-authorized booking through constrained transaction; no general anonymous table policy
Lead events	Read own	Read own	System-created immutable audit events; no arbitrary member UPDATE/DELETE or fabricated delivery timestamps
Intake credentials	None	Manage own via authenticated, owner-checked functions; no hash/plaintext exposure	Server verifies hash before privileged intake action
Booking tokens	No raw token/hash reads	Authorized issuance/revocation through functions	Token-scoped minimal lookup/action; reject expired/revoked/invalid tokens
Contact preferences	Read minimum necessary own-company state	Manage lawful configuration; cannot erase opt-out via arbitrary edits	Verified callbacks record STOP; only consent-validated server actions send
Jobs / attempts	Own-company safe read projection	Own-company safe read and authorized retry request	Only verified processors/callbacks mutate authoritative status, leases and provider IDs
Calendar connections / busy intervals	Own-company safe connection/availability read	Authorize/disconnect own-company reps	Connector callbacks map verified account to company/rep; credential storage remains server-only
Apply USING and WITH CHECK to tenant-scoped writes. Prevent moving a record between companies. Database constraints must enforce parent/child tenant consistency even when the privileged server bypasses RLS. Where an owner can manage hidden/revoked records, provide an owner SELECT path as well as write policies.

Grant policy: grant authenticated users only the operations allowed by policies; grant service role the operations needed for verified system work; grant no anonymous access to private tables. For sensitive columns, use safe projections plus column-level restrictions or a separate protected table—RLS filters rows, not columns.

Function safety: authenticated calls validate the session and membership/role using the caller-scoped client before privileged work. Existing server functions have bearer middleware support; preserve it. Database helper/booking functions need fixed search paths, narrow execute permissions and explicit caller/tenant validation. Keep ordinary reads under user RLS.

Auth cleanup: current /app protection lives in AppShell with client-effect redirects and repeated auth observers. Move protected screens into the integration-managed authenticated route layout while preserving /app URLs and public /auth, /book/$token and intake behavior. Centralize identity-transition handling; preserve intended destinations only as sanitized same-origin paths. Verify configured sign-up/sign-in settings rather than inferring them from source.

## Edge cases hardened

| Case | Before | After |
|---|---|---|
| Empty / first-run state | Intake Before: empty/whitespace name, invalid/non-SMS number, optional missing email, oversized JSON/form body, spam trap, unsupported content type, duplicate submission, expired/rotated key, consent absent, public form abuse. CORS is not authentication and a secret embedded in HTML is not private. | Intake After |
| Bad / malicious input | Booking Before: simultaneous SMS/page/voice attempts, replayed confirmation click, stale offered time, past/malformed timestamp, lead already booked, database availability failure, conflicting external event, no active reps or rep removed mid-flow. Reject conflict with refreshed slots; do not present query failure as empty availability. | Booking After |
| Failure / offline | Time rules Before: invalid timezone, user in a different timezone, spring-forward/nonexistent time, fall-back/ambiguous time, exact opening/closing boundaries, weekends/holidays, duration exceeding daily hours, zero/negative duration. Holiday/buffer rules need a scope decision rather than invented behavior. | Time Rules After |

## Stress test results

_What you threw at it, and what held / broke._

Spam Click: click submit 5× rapidly, does it create 5 duplicate records, or hold the line? Held the line, only one user added
