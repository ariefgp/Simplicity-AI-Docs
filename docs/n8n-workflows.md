# n8n Workflow Specs: WF-B, WF-C, WF-D, WF-E

## Conventions

[GHL-AUTH] on any HTTP Request to services.leadconnectorhq.com:
- Authentication: Generic Credential Type > Header Auth > credential "GHL PIT"
  (Name: Authorization, Value: Bearer <token>)
- Send Headers ON > Header: Name Version, Value 2021-07-28
- Options > Timeout: 5000

[RETELL-AUTH] on any HTTP Request to api.retellai.com:
- Same pattern, credential "Retell API"

[JSON-BODY]:
- Send Body ON, Body Content Type JSON, Specify Body: Using JSON,
  field in Expression mode when it contains {{ }}

All webhook trigger nodes: Authentication None (Header Auth added in
hardening phase). All Respond nodes: Response Code 200.

---

## WF-B: book-appointment

Flow: Webhook > Clinic Config > Extract Args > Get Contact >
IF Already Booked > (true) Set Already Booked > Respond Already Booked /
(false) Create Appointment > (success) Add Booked Tag > Move Opportunity >
Format Success > Respond OK / (error) Alert Placeholder > Set Fail >
Respond Fail. Get Contact error output also goes to Set Fail.

Node 1 Webhook: POST, path retell/book-appointment, Respond: Using
'Respond to Webhook' Node.

Node 2 Clinic Config (Set, Manual Mapping, Include Other Input Fields ON),
String fields: calendar_id, location_id, booked_stage_id, pipeline_id,
clinic_tz. Values from docs/ids.md (booked_stage_id = stage_booked).

Node 3 Extract Args (Code, Run Once for All Items):
const body = $('Webhook').first().json.body;
const args = body.args || {};
return [{ json: {
  contactId: args.contact_id,
  slotStart: args.slot_start,
  opportunityId: args.opportunity_id || '',
  notes: args.notes || '',
  callId: body.call?.call_id || 'unknown',
}}];

Node 4 Get Contact (HTTP): GET
https://services.leadconnectorhq.com/contacts/{{ $json.contactId }}
[GHL-AUTH]. Settings: On Error Continue (using error output).
Success > Node 5. Error > Set Fail.

Node 5 IF Already Booked: Left (Expression)
{{ ($json.contact?.tags || []).includes('booked') }} Boolean is true.
True > Set Already Booked. False > Create Appointment.

Node 6a Set Already Booked (Set): response = "This lead already has an
appointment booked. Tell them their appointment is confirmed and the
clinic will see them soon."
Node 6b Respond Already Booked: Respond With First Incoming Item, 200.

Node 7 Create Appointment (HTTP): POST
https://services.leadconnectorhq.com/calendars/events/appointments
[GHL-AUTH] [JSON-BODY]:
{
  "calendarId": "{{ $('Clinic Config').first().json.calendar_id }}",
  "locationId": "{{ $('Clinic Config').first().json.location_id }}",
  "contactId": "{{ $('Extract Args').first().json.contactId }}",
  "startTime": "{{ $('Extract Args').first().json.slotStart }}",
  "appointmentStatus": "confirmed",
  "title": "AI Booked Assessment"
}
On Error Continue (using error output). Success > Node 8. Error > Node 12.

Node 8 Add Booked Tag (HTTP): POST
https://services.leadconnectorhq.com/contacts/{{ $('Extract Args').first().json.contactId }}/tags
[GHL-AUTH] [JSON-BODY]: { "tags": ["booked"] }
On Error Continue (using regular output).

Node 9 Move Opportunity (HTTP): PUT
https://services.leadconnectorhq.com/opportunities/{{ $('Extract Args').first().json.opportunityId }}
[GHL-AUTH] [JSON-BODY]:
{ "pipelineStageId": "{{ $('Clinic Config').first().json.booked_stage_id }}" }
On Error Continue (using regular output). Rationale: after the appointment
exists, nothing may derail the success response; WF-C repeats stage+tag.

Node 10 Format Success (Code):
const slot = $('Extract Args').first().json.slotStart;
const tz = $('Clinic Config').first().json.clinic_tz;
const human = new Intl.DateTimeFormat('en-US', {
  weekday: 'long', month: 'long', day: 'numeric',
  hour: 'numeric', minute: '2-digit', timeZone: tz,
}).format(new Date(slot));
return [{ json: {
  response: `Booked successfully for ${human}. Confirm the details to the lead, then proceed to transfer.`
}}];

Node 11 Respond OK: First Incoming Item, 200.

Node 12 Alert Placeholder (Set): alert = (Expression)
"Booking failed for contact {{ $('Extract Args').first().json.contactId }}
slot {{ $('Extract Args').first().json.slotStart }}"
(Replace with Gmail/Slack node in hardening phase.)

Node 13a Set Fail (Set): response = "Booking failed. Apologize briefly and
tell the lead the clinic will follow up shortly to lock in their time."
Node 13b Respond Fail: First Incoming Item, 200.

---

## WF-C: post-call processor

Flow: Webhook > Verify Signature > IF Analyzed > (true) Extract >
Clinic Config > Route Outcome (Switch, 7 outputs) > branch HTTP nodes >
all branches merge into Update Contact Fields > Add Note.
IF Analyzed false > NoOp.

Node 1 Webhook: POST, path retell/post-call, Respond: Immediately.

Node 2 Verify Signature (Code):
const crypto = require('crypto');
const apiKey = 'YOUR_RETELL_API_KEY';
const signature = $('Webhook').first().json.headers['x-retell-signature'];
const payload = JSON.stringify($('Webhook').first().json.body);
const expected = crypto.createHmac('sha256', apiKey).update(payload).digest('hex');
if (!signature || signature !== expected) {
  throw new Error('Invalid Retell signature');
}
return $input.all();
If require('crypto') is unavailable on n8n Cloud: omit this node, record as
hardening debt.

Node 3 IF Analyzed: Left {{ $json.body.event }} String is equal to
call_analyzed. False > No Operation node.

Node 4 Extract (Code):
const b = $('Webhook').first().json.body;
const call = b.call || {};
const analysis = call.call_analysis || {};
const custom = analysis.custom_analysis_data || {};
let outcome = custom.call_outcome || '';

const dr = call.disconnection_reason || '';
const noAnswerReasons = ['dial_no_answer', 'dial_busy', 'dial_failed', 'voicemail_reached'];
if (noAnswerReasons.includes(dr)) outcome = 'did_not_answer';

const valid = ['appointment_booked','transferred_to_office','follow_up',
               'not_interested','dnd_stop','did_not_answer'];
if (!valid.includes(outcome)) outcome = 'unknown';

const labelMap = {
  appointment_booked: 'Appointment Booked',
  transferred_to_office: 'Call Transferred to Office',
  follow_up: 'Follow-up',
  not_interested: 'Not Interested',
  dnd_stop: 'DND/Stop',
  did_not_answer: 'Did Not Answer',
  unknown: 'Follow-up',
};

return [{ json: {
  outcome,
  outcomeLabel: labelMap[outcome],
  callId: call.call_id || '',
  contactId: call.metadata?.contact_id || '',
  opportunityId: call.metadata?.opportunity_id || '',
  summary: custom.call_summary || analysis.call_summary || '',
  bookedTime: custom.booked_time || '',
}}];

Node 5 Clinic Config (Set, Include Other Input Fields ON), String fields
with values from docs/ids.md: location_id, pipeline_id, stage_booked,
stage_engaged, stage_followup, stage_dnd, stage_no_answer,
field_call_outcome, field_call_summary, field_retell_call_id.

Node 6 Route Outcome (Switch, Mode Rules): six rules, each
Left (Expression) {{ $('Extract').first().json.outcome }} is equal to
[value2 plain text]: appointment_booked, transferred_to_office, follow_up,
not_interested, dnd_stop, did_not_answer. Rename Output ON per rule.
Options > Fallback Output: Extra Output (7th, for unknown).

Branch HTTP nodes, all [GHL-AUTH] [JSON-BODY], On Error Continue (regular
output):
- appointment_booked: PUT /opportunities/{{ $('Extract').first().json.opportunityId }}
  body { "pipelineStageId": "{{ $('Clinic Config').first().json.stage_booked }}" }
  then POST /contacts/{{ $('Extract').first().json.contactId }}/tags
  body { "tags": ["booked"] }
- transferred_to_office: same stage PUT (stage_booked), then tags
  { "tags": ["transferred-office"] }
- follow_up: stage PUT (stage_followup)
- not_interested: stage PUT (stage_followup), then tags
  { "tags": ["not-interested"] }
- dnd_stop: stage PUT (stage_dnd), then tags { "tags": ["dnd-stop"] },
  then PUT /contacts/{{ $('Extract').first().json.contactId }}
  body { "dnd": true }
- did_not_answer: stage PUT (stage_no_answer)
- fallback: stage PUT (stage_followup), then Set node alert =
  "Unknown outcome for call {{ $('Extract').first().json.callId }}"

Node 7 Update Contact Fields (HTTP), all branch ends connect here: PUT
https://services.leadconnectorhq.com/contacts/{{ $('Extract').first().json.contactId }}
[GHL-AUTH] [JSON-BODY]:
{
  "customFields": [
    { "id": "{{ $('Clinic Config').first().json.field_call_outcome }}", "value": "{{ $('Extract').first().json.outcomeLabel }}" },
    { "id": "{{ $('Clinic Config').first().json.field_call_summary }}", "value": "{{ $('Extract').first().json.summary }}" },
    { "id": "{{ $('Clinic Config').first().json.field_retell_call_id }}", "value": "{{ $('Extract').first().json.callId }}" }
  ]
}
On Error Continue (regular output).

Node 8 Add Note (HTTP): POST
https://services.leadconnectorhq.com/contacts/{{ $('Extract').first().json.contactId }}/notes
[GHL-AUTH] [JSON-BODY]:
{ "body": "AI call ({{ $('Extract').first().json.outcome }}): {{ $('Extract').first().json.summary }}" }

---

## WF-D: initiate-call

Flow: Webhook > Clinic Config > Validate Phone > IF Valid > (false)
Tag Invalid > NoOp / (true) Get Contact > IF Stopped > (true) NoOp /
(false) Check Window > IF In Window > (false) NoOp / (true) Create Call >
(success) Record Attempt / (error) Alert Placeholder.

Node 1 Webhook: POST, path ghl/initiate-call, Respond: Immediately.

Node 2 Clinic Config (Set, Include Other Input Fields ON): from_number,
voice_agent_id, clinic_tz, field_attempt_count. Values from docs/ids.md.

Node 3 Validate Phone (Code):
const b = $('Webhook').first().json.body;
const raw = (b.phone || '').replace(/[\s\-().]/g, '');
const e164 = /^\+[1-9]\d{7,14}$/;
const phone = raw.startsWith('+') ? raw : (raw.length === 10 ? '+1' + raw : raw);
return [{ json: {
  valid: e164.test(phone),
  phone,
  contactId: b.contact_id,
  opportunityId: b.opportunity_id || '',
  firstName: b.first_name || '',
  lastName: b.last_name || '',
  service: b.service_of_interest || '',
  attempt: String(b.attempt || '1'),
}}];
Note: the +1 default is US-specific; flag in setup guide for non-US clones.

Node 4 IF Valid: {{ $json.valid }} Boolean is true. False > Tag Invalid
(HTTP POST /contacts/{{ $json.contactId }}/tags [GHL-AUTH] [JSON-BODY]
{ "tags": ["invalid-phone"] }) > NoOp. True > Node 5.

Node 5 Get Contact (HTTP): GET
https://services.leadconnectorhq.com/contacts/{{ $json.contactId }}
[GHL-AUTH].

Node 6 IF Stopped: Left (Expression)
{{ ['booked','dnd-stop','not-interested','transferred-office'].some(t => ($json.contact?.tags || []).includes(t)) }}
Boolean is true. True > NoOp. False > Node 7.

Node 7 Check Window (Code):
const tz = $('Clinic Config').first().json.clinic_tz;
const hour = parseInt(new Intl.DateTimeFormat('en-US',
  { hour: 'numeric', hour12: false, timeZone: tz }).format(new Date()));
return [{ json: { ...$('Validate Phone').first().json, inWindow: hour >= 9 && hour < 21 } }];
Then IF In Window: {{ $json.inWindow }} Boolean is true. False > NoOp.
True > Node 8.

Node 8 Create Call (HTTP): POST https://api.retellai.com/v2/create-phone-call
[RETELL-AUTH] [JSON-BODY]:
{
  "from_number": "{{ $('Clinic Config').first().json.from_number }}",
  "to_number": "{{ $json.phone }}",
  "override_agent_id": "{{ $('Clinic Config').first().json.voice_agent_id }}",
  "retell_llm_dynamic_variables": {
    "first_name": "{{ $json.firstName }}",
    "last_name": "{{ $json.lastName }}",
    "service_of_interest": "{{ $json.service }}",
    "contact_id": "{{ $json.contactId }}",
    "opportunity_id": "{{ $json.opportunityId }}",
    "attempt_number": "{{ $json.attempt }}"
  },
  "metadata": {
    "contact_id": "{{ $json.contactId }}",
    "opportunity_id": "{{ $json.opportunityId }}",
    "attempt": "{{ $json.attempt }}"
  }
}
Settings: On Error Continue (using error output); Retry On Fail ON,
Max Tries 2, Wait Between Tries 5000.
Success > Node 9. Error > Alert Placeholder (Set) alert =
"Retell call failed for {{ $json.contactId }}".

Node 9 Record Attempt (HTTP): PUT
https://services.leadconnectorhq.com/contacts/{{ $('Validate Phone').first().json.contactId }}
[GHL-AUTH] [JSON-BODY]:
{ "customFields": [ { "id": "{{ $('Clinic Config').first().json.field_attempt_count }}", "value": "{{ $('Validate Phone').first().json.attempt }}" } ] }

---

## WF-E: SMS chat bridge

Prerequisite: GHL custom field retell_chat_id exists; its ID recorded in
docs/ids.md as field_chat_id.

Flow: Webhook > Clinic Config > Extract > IF Stop Intent > (true) Set DND >
Tag DND > NoOp / (false) Get Contact > IF DND > (true) NoOp / (false)
Read Chat ID > IF Has Chat > (true) Create Completion / (false)
Create Chat > Save Chat ID > Normalize Chat ID > Create Completion >
Extract Reply > IF Has Reply > (true) Send SMS / (false) NoOp.

Node 1 Webhook: POST, path ghl/inbound-sms, Respond: Immediately.

Node 2 Clinic Config (Set, Include Other Input Fields ON): chat_agent_id,
location_id, field_chat_id.

Node 3 Extract (Code):
const b = $('Webhook').first().json.body;
return [{ json: {
  contactId: b.contact_id,
  message: (b.message || '').trim(),
  firstName: b.first_name || '',
  service: b.service_of_interest || '',
}}];

Node 4 IF Stop Intent: Left (Expression)
{{ ['stop','unsubscribe','quit','cancel','end','stopall','berhenti'].includes($json.message.toLowerCase().replace(/[^a-z]/g,'')) }}
Boolean is true.
True > Set DND (HTTP PUT /contacts/{{ $json.contactId }} [GHL-AUTH]
[JSON-BODY] { "dnd": true }) > Tag DND (HTTP POST
/contacts/{{ $('Extract').first().json.contactId }}/tags [JSON-BODY]
{ "tags": ["dnd-stop"] }) > NoOp. No reply is sent.
False > Node 5.

Node 5 Get Contact (HTTP): GET
https://services.leadconnectorhq.com/contacts/{{ $json.contactId }}
[GHL-AUTH].

Node 6 IF DND: {{ ($json.contact?.tags || []).includes('dnd-stop') }}
Boolean is true. True > NoOp. False > Node 7. (Only dnd-stop blocks chat;
booked leads may still ask questions.)

Node 7 Read Chat ID (Code):
const contact = $('Get Contact').first().json.contact || {};
const fieldId = $('Clinic Config').first().json.field_chat_id;
const cf = (contact.customFields || []).find(f => f.id === fieldId);
return [{ json: { ...$('Extract').first().json, chatId: cf?.value || '' } }];
Then IF Has Chat: {{ $json.chatId }} String is not empty.
True > Node 9. False > Node 8a.

Node 8a Create Chat (HTTP): POST https://api.retellai.com/create-chat
[RETELL-AUTH] [JSON-BODY]:
{
  "agent_id": "{{ $('Clinic Config').first().json.chat_agent_id }}",
  "retell_llm_dynamic_variables": {
    "first_name": "{{ $json.firstName }}",
    "service_of_interest": "{{ $json.service }}",
    "contact_id": "{{ $json.contactId }}"
  }
}

Node 8b Save Chat ID (HTTP): PUT
https://services.leadconnectorhq.com/contacts/{{ $('Extract').first().json.contactId }}
[GHL-AUTH] [JSON-BODY]:
{ "customFields": [ { "id": "{{ $('Clinic Config').first().json.field_chat_id }}", "value": "{{ $('Create Chat').first().json.chat_id }}" } ] }

Node 8c Normalize Chat ID (Code):
return [{ json: { ...$('Read Chat ID').first().json, chatId: $('Create Chat').first().json.chat_id } }];
> Node 9.

Node 9 Create Completion (HTTP): POST
https://api.retellai.com/create-chat-completion [RETELL-AUTH] [JSON-BODY]:
{ "chat_id": "{{ $json.chatId }}", "content": "{{ $('Extract').first().json.message }}" }

Node 10 Extract Reply (Code):
const resp = $json;
const msgs = (resp.messages || []).filter(m => m.role === 'agent');
const reply = msgs.map(m => m.content).join(' ').trim();
return [{ json: { reply, contactId: $('Extract').first().json.contactId } }];

Node 11 IF Has Reply: {{ $json.reply }} String is not empty.
True > Node 12. False > NoOp.

Node 12 Send SMS (HTTP): POST
https://services.leadconnectorhq.com/conversations/messages
[GHL-AUTH] [JSON-BODY]:
{ "type": "SMS", "contactId": "{{ $json.contactId }}", "message": "{{ $json.reply }}" }

---

## Tests (a workflow is done only when its tests pass)

WF-B: POST test payload
{ "call": {"call_id":"t1"}, "args": { "contact_id": "<test contact>",
"slot_start": "<ISO slot from WF-A>", "opportunity_id": "<test opp>" } }
Verify: appointment in GHL calendar, tag booked, opportunity at Booked
stage. Repeat identical payload: already-booked response, no duplicate.

WF-C: real Retell test call ending "not interested": stage Follow-up, tag
not-interested, fields and note written. Ignored call: outcome forced to
did_not_answer.

WF-D: POST with own phone: Maya calls. With tag booked on contact: no
call. With phone "12345": no call, tag invalid-phone applied.

WF-E: message "what are your hours?": chat created in Retell,
retell_chat_id saved on contact, SMS reply arrives via GHL. Follow-up
"can I book Tuesday?": WF-A execution appears.
