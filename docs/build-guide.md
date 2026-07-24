# Simplicity AI Build Guide
GoHighLevel + Retell AI + n8n. Verified July 2026.

Platform facts: GHL V1 API is end-of-life; use API V2 via Private
Integration Token, base URL services.leadconnectorhq.com, header
Version: 2021-07-28. Retell outbound via POST /v2/create-phone-call with
retell_llm_dynamic_variables and metadata. Retell numbers use weighted
agent lists (outbound_agents). Retell custom functions POST to n8n and
expect a fast response string. Post-call webhooks (call_analyzed) carry
x-retell-signature, 10s timeout, 3 retries.

## Phase 1: GHL Foundation
1.1 Sub-account: clinic timezone in Business Profile drives all
9 AM to 9 PM windows.
1.2 PIT: Settings > Private Integrations. Scopes: contacts r/w,
opportunities r/w, calendars read, calendar events r/w, conversations
read, conversation messages write, locations read. Store only in n8n
credentials. Smoke test:
curl https://services.leadconnectorhq.com/locations/{locationId}
  -H "Authorization: Bearer PIT" -H "Version: 2021-07-28"
1.3 Custom fields (contact): call_outcome (dropdown, 6 labels),
call_summary (multi-line), retell_call_id (text), call_attempt_count
(number), service_of_interest (dropdown), retell_chat_id (text, for WF-E).
Fields are created in Settings; forms borrow them via Add Object Fields.
1.4 Pipeline: New Lead, Engaged/Answered (ours: "Contacted"), Booked
Appointment, Follow-up, DND / Stop, Did Not Answer. IDs via
GET /opportunities/pipelines?locationId=...
1.5 Calendar: one service calendar; verify slots render in the GHL
booking widget before wiring the API.
1.6 Landing page + form: First/Last Name, Phone (required), Email,
Service of Interest. Duplicate contacts OFF. Thank-you step says the
assistant calls in about a minute.
1.7 Tags: ai-calling-active, booked, dnd-stop, not-interested,
nurture-45, invalid-phone, transferred-office. Create in Settings > Tags;
only select (never type) tags in workflows.

## Phase 2: Retell Agents
2.1 Voice agent: Conversation Flow. Name {Clinic} - Outbound Booking -
Voice. Dynamic variables: first_name, last_name, service_of_interest,
contact_id, opportunity_id, attempt_number. Canvas spec in
docs/retell-agent.md.
2.2 Functions (per agent): check_availability and book_appointment,
POSTing to the n8n production webhook URLs. Schemas in
docs/retell-agent.md. Talk While Waiting ON with static filler; Wait for
Result ON. Transfer via Call Transfer node (not Agent Transfer).
2.3 Post-Call Data Extraction fields: call_outcome (selector, 6 values
with strict definitions), call_summary (text), booked_time (text).
2.4 Agent webhook: n8n WF-C production URL, event call_analyzed.
2.5 Phone number: assign voice agent under outbound agents list. Pre-call
SMS states the Retell number explicitly since GHL sends the SMS from a
different number.
2.6 Chat agent: separate agent, SMS rules (max 2 short sentences, one
question, instant STOP acknowledgment), same functions, same extraction
fields. Lives behind the API only (WF-E), no number assignment.

## Phase 3: n8n
Credentials: "GHL PIT" and "Retell API" (Header Auth). Config via
per-workflow Clinic Config Set nodes (Variables unavailable on this
plan). Full specs: docs/n8n-workflows.md. WF-A summary: webhook >
config > date range (epoch ms) > GET /calendars/{id}/free-slots > format
3 to 5 slots as sentence + ISO array > respond { response, slots };
error branch responds static fallback with empty slots, HTTP 200.

## Phase 4: GHL Workflows
W1 Five call attempts: Form Submitted trigger > dedupe guard (tags) >
create opportunity at New Lead > tag ai-calling-active > phone check >
pre-call SMS > wait 1 min > business-hours gate > stop-rule gate >
webhook to WF-D > configurable wait > repeat gates+webhook for attempts
2 to 5 > after 5 unbooked: remove ai-calling-active, tag nurture-45,
enroll in W4.
W2 SMS chat: Customer Replied (SMS) trigger > if dnd-stop end > webhook
to WF-E. Enable GHL native STOP compliance.
W3 Pipeline movement: Customer Replied trigger > if stage New Lead and
not booked/dnd > move to Engaged. Booked stage always wins; guard ends
workflow if already Booked.
W4 45-day nurture: 6 sends, 7 days apart, gates before every send,
message text generated via n8n (offer + week number), replies flow to W2,
exit on booked/dnd-stop/not-interested, remove nurture-45 on exit.
W5 Confirmation + reminders: Appointment Booked trigger > remove from
W1/W2-promo/W4 > immediate confirmation SMS+email > reminders 24h and 2h
before (editable offsets).

## Phase 5: Hardening, Acceptance, Handover
Hardening: Header Auth secrets on all webhooks (different secret per
caller system), Retell signature verification on WF-C, n8n Error Workflow
alerting, DND enforced at three layers (GHL DND flag, tag gates, n8n
guards), no secrets in prompts or workflow JSON.
Acceptance test: form submit (one contact, one opportunity, no dupes on
resubmit) > pre-call SMS > call in ~1 min > ignored call logs Did Not
Answer and schedules attempt 2 > answered call books real calendar
appointment, moves stage, attempts transfer > attempts 3 to 5 never fire
after booking > confirmation and reminders arrive > STOP silences
everything > "not interested" stops nurture > broken calendar ID makes
agent refuse to promise a time and fires internal alert.
Setup guide deliverable: list every per-clinic value (GHL: sub-account,
PIT, location/calendar/pipeline/stage/field IDs, form options, timezone,
call gap, reminder offsets, 7 tag names; Retell: agent names, CLINIC
CONFIG blocks, transfer number, phone number; n8n: duplicated workflows,
new webhook URLs pasted into Retell and GHL). Cloning order: GHL snapshot
> new PIT > duplicate n8n + swap credentials > duplicate Retell agents +
edit config > paste new webhook URLs > run acceptance test.

## Key API reference
Free slots: GET /calendars/{calendarId}/free-slots?startDate={ms}&endDate={ms}&timezone={tz}
Create appointment: POST /calendars/events/appointments
  { calendarId, locationId, contactId, startTime (ISO+offset),
    appointmentStatus: "confirmed" }
Tags: POST /contacts/{id}/tags { "tags": ["..."] }
Stage: PUT /opportunities/{id} { "pipelineStageId": "..." }
Contact fields: PUT /contacts/{id} { "customFields": [{id, value}], "dnd": bool }
Note: POST /contacts/{id}/notes { "body": "..." }
Send SMS: POST /conversations/messages { "type":"SMS", contactId, message }
Custom field IDs: GET /locations/{locationId}/customFields
Pipelines: GET /opportunities/pipelines?locationId=...
Retell call: POST https://api.retellai.com/v2/create-phone-call
Retell chat: POST /create-chat, POST /create-chat-completion
