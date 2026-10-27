# Living PRD

> Module 4 · Production Specs. Refactor for readability; extract a living PRD that stays true as the build evolves.

## Problem

_What user problem does this solve? Tie to the validated hypothesis._

Hypothesis: The first contractor to reach a lead wins the job. Homeowners send the same request to several contractors at once, and the one that answers while they are still deciding gets the appointment.

How far it has been validated: the prototype proves the mechanism, not the market.

Proven in the prototype (Real): a form submission gets a first text logged within seconds. The app records time-to-first-text for every lead. The AI texting assistant booked a real slot from a free-text reply. Booking writes an appointment and sends confirmations, which are logged.
Not yet proven (open): that contractors who reply faster actually book more jobs. The prototype has no real homeowners, real texts or calls, or conversion data from a live contractor. The Speed-to-lead page is built to measure this, but so far it only has test-lead data.
Problem statement: Home-services contractors lose booked jobs because nobody responds to web leads within minutes, especially after hours or when the office is busy. Contractors need every lead contacted instantly and pushed to a booked appointment on a rep's calendar, without anyone on staff having to do it.

## Users & jobs

- **Primary user:** Primary user: the owner or office manager of a small-to-mid home-services contractor (HVAC, roofing, plumbing) who gets leads from a website form.
- **Job to be done:** Job to be done: "When a lead fills out my website form, contact them right away and get them booked with one of my reps, so I win the job before a competitor calls them back."

## Scope

- **In:** In scope (this prototype)
Area	Status
Sign up / sign in (email, Google, Lovable) and a company workspace	Real
Multiple companies on one product, with data kept separate per company	Real
Lead intake address plus a copy-paste form snippet (secret key, spam trap)	Real
"Send test lead" from the dashboard	Real
Instant first text to the lead with a booking link	Mocked: logged on the timeline, not sent
AI voice call right after the text	Mocked: a "dialed" entry is logged, no call is placed
AI texting assistant that offers open times and books	Real AI and real booking. Incoming texts come only from the "Reply as the lead" box
Taking turns assigning reps to new leads	Real
Open times calculated from business hours, appointment length and existing bookings	Real (with tests)
Public self-booking page per lead	Real
Confirmation text and email to the lead and the rep	Mocked: logged only
Leads list with search, status filter, side panel and live updates	Real
Appointments list	Real
Settings → Rep calendars tab (connect button plus rep day view)	Day view is Real; the connect button is Mocked
Speed-to-lead page (average time to text, under-10s %, booking rate, 14-day status chart)	Real on test data
Lost-lead recovery queue with Text again / Call again	Queue is Real; the retries are Mocked sends
- **Out (explicitly):** Live text and voice provider (for example Twilio + ElevenLabs) and incoming texts from real phones
Live email sending (needs a verified sender domain)
Real Google Calendar sync and reading reps' existing busy times
Rescheduling or cancelling from a link; reminders before appointments
Automatic "no answer" detection (nothing sets that status yet) and automatic retry schedules
Team invites and roles beyond the creator as owner
Billing, CRM integrations, text-message consent and opt-out handling, call recording

## Requirements

| # | Requirement | Priority | Acceptance criteria |
|---|---|---|---|
| 1 | Instant first text | Must | 95% of valid leads get a text sent within 10s of submission. Time-to-text is stored per lead. |
| 2 | AI call right after the text | Must | A call is placed to the submitted number within 30s of the text. The outcome (answered, voicemail, no answer) is logged. A booking made on the call uses the same booking path. |

## Data & events

_What gets stored, what gets tracked._

Records (each one belongs to a company):

Record	Key fields
Company	name, intake key and secret, time zone, opening and closing hours, appointment length, test mode
Company member	user, role (owner / member)
Service	name
Rep	name, email, phone, active, last assigned
Lead	name, phone, email, service, address, notes, status (new → texted → calling → booked / no_answer), assigned rep, booking link code, first text time, call start time
Appointment	lead, rep, start, end, booked via (text / call / page), calendar event link (empty for now)
Lead event	kind, body, details (recipient, audience, test mode, channel)
Timeline events written by the prototype:

Event	When	Real / mocked
sms_out (to lead or rep)	first text, assistant replies, confirmations, recovery text	Logged; nothing sent
sms_in	"Reply as the lead" box	Simulated input
call	after the first text; recovery call	Logged; nothing dialed
email (to lead or rep)	booking confirmations	Logged; nothing sent
booked	appointment created	Real
calendar	after booking	Placeholder note; no calendar event
retry	Text again / Call again in Recovery	Real log entry; the send itself is mocked
Metrics calculated from this data: time to first text (first text time minus submission time), % under 10s, booking rate (booked ÷ leads), and leads by status per day.

## Open questions

Hypothesis proof: which pilot contractor and which before/after measure (booking rate against their current response time) will count as validation?
Text and voice provider: Twilio + ElevenLabs, or something else? Which phone number setup do we use for each contractor?
Consent and compliance: what text-message consent wording goes on contractors' forms, and who owns business-texting registration?
Voice agent behaviour: how many call attempts, voicemail script, handing off to a live person, and recording or disclosure rules.
No-answer definition: when does a lead become "no answer" (call outcome, X hours without a reply)? Should recovery retries run automatically on a schedule?
Calendar: Google only, or Outlook too? Does a rep's existing calendar block slots, and who can change a booked event?
Booking rules: service-specific appointment lengths, travel buffers, service areas, same-day cutoffs?
Leads outside business hours: text immediately but call in the morning, or both right away?
Teams: invite flow, a rep-only login to see their own day, and permissions?
Email sender: each contractor's own domain or a shared sender?
