# Per-Clinic Setup and Clone Guide

Deliverable for spec section 12: where clinic details, prompts, IDs, phone
numbers, calendars, and credentials change for each new client.

This system is a template. The template clinic is "Comfort Care Physiotherapy"
(Frisco TX, agent persona "Maya"). To onboard a new clinic you copy the same
three-system structure (GoHighLevel + Retell + n8n) and change a fixed set of
values. Nothing about the LOGIC changes between clinics; only content and IDs.

The golden rule for finding what to change: in n8n, every workflow has a Set
node named "Clinic Config" (WF-A's is the first Set node after the webhook).
Those nodes are the single place per-workflow IDs live. Clinic prose lives in
the two Retell prompts. Secrets live only in n8n credentials.

--------------------------------------------------------------------------
## 0. What is shared vs per-clinic

Per-clinic (changes every time):
- All GHL IDs: location, pipeline, 6 stages, 6 custom fields, calendar, form.
- Clinic content: name, address, hours, services, offer, pricing, FAQs,
  qualification, disqualification, emergency guidance, disclaimer.
- Timezone.
- Phone numbers: the Retell outbound number and the office transfer number.
- The two Retell agents (voice + chat) and their prompts.
- The GHL landing page, form, and the five GHL automation workflows.
- Credentials: GHL Private Integration Token, Retell API key.

Shared / do NOT change per clinic:
- The five n8n webhook PATHS. They are contracts with Retell and GHL configs:
  retell/check-availability, retell/book-appointment, retell/post-call,
  ghl/initiate-call, ghl/inbound-sms.
- The workflow logic, node wiring, and code nodes.
- The call_outcome enum -> GHL picklist label map (WF-C Extract code).
- The seven tag strings (see docs/ids.md), unless the clinic wants different
  tags, in which case change them everywhere they appear.

--------------------------------------------------------------------------
## 1. Reference: the template clinic's values

These are the current template IDs (from docs/ids.md). Replace each with the
new clinic's equivalent. Keep docs/ids.md updated per clinic, or fork it.

n8n instance: https://YOUR_N8N_HOST
n8n workflow IDs (this instance):
- WF-A check-availability: txqpWjZi7KqsPbH4   (path retell/check-availability)
- WF-B book-appointment:   r78iS4jBvqZZYnrI   (path retell/book-appointment)
- WF-C post-call:          rYXGVDcCIARtmN5T   (path retell/post-call)
- WF-D initiate-call:      GNL3tFcyHzccxbNl   (path ghl/initiate-call)
- WF-E inbound-sms:        GWYO7x5E0bSp43Gv   (path ghl/inbound-sms)

n8n credentials (this instance):
- GHL:    "Bearer Auth account 2", type Bearer Auth,  id 9OFLLpyJRn9RGMul
- Retell: "Retell API",            type Header Auth,  id NUR7UGXDR9tzLKEG

GHL:  location 1nw8goHWjkKjkpdfUWIQ, pipeline QqHBuuQtTOYwQ4ZohYqf,
      calendar cleS2YdMFlgsDHnZpch4.
Retell: voice agent_027b4f12a2b6520c3e450fb6c3 (flow
      conversation_flow_a76cb9158559), chat agent_a7739f2c1375fc9b1051456b74
      (LLM llm_631bc396538dfb70df2ad9dd8838), from_number YOUR_RETELL_NUMBER,
      transfer number OFFICE_TRANSFER_NUMBER.

--------------------------------------------------------------------------
## 2. Clone checklist, in order

Do the steps in this order; later steps depend on IDs created earlier.

### Step A. GHL foundation (UI)
1. Create or use the clinic's GHL sub-account. Set the account timezone.
2. In Settings > Custom Values / location, set duplicate prevention:
   allowDuplicateContact OFF, allowDuplicateOpportunity OFF.
3. Pipeline: create a pipeline with these six stages, in this meaning order:
   New Lead, Contacted (engaged), Booked Appointment, Follow-up,
   Did Not Answer, DND / Stop. Record each stage ID.
4. Custom fields (Contact): create these six and record each ID:
   - Call Outcome (Single option) with picklist EXACTLY:
     Follow-up, Not Interested, DND/Stop, Appointment Booked,
     Call Transferred to Office, Did Not Answer.
   - Call Summary (Multi line)
   - Retell Call ID (Single line)
   - Call Attempt Count (Number)
   - Service of Interest (Single option) with the clinic's offers
   - Retell Chat ID (Single line, key contact.retell_chat_id)
5. Tags: the workflows use ai-calling-active, booked, dnd-stop,
   not-interested, nurture-45, invalid-phone, transferred-office. Create them
   or keep the defaults.
6. Calendar: create the consultation/assessment calendar. Record its ID.
7. Landing page + form: build one funnel/landing page with a lead form that
   has at least first name, last name, phone, email, and service of interest
   (mapped to the Service of Interest custom field). On submit it must
   create/update the contact and create an opportunity at New Lead. Confirm
   the form's Add Object Fields mapping writes service_of_interest.

### Step B. Retell agents (dashboard + API)
The clinic's knowledge lives in TWO prompts. Edit both.
1. Voice agent: clone the Conversation Flow agent. Edit the flow's
   global_prompt CLINIC CONFIG block: clinic name, address, hours, services,
   offer, pricing, qualification questions, disqualification rules, emergency
   guidance, transfer instruction, disclaimer, timezone. Set the agent
   timezone to the clinic timezone. In the Transfer node, set the office
   number. Assign the clinic's Retell number as the agent's outbound number.
2. Chat agent: clone the chat agent + its Retell LLM. Edit the LLM
   general_prompt CLINIC CONFIG block with the same clinic content adapted for
   SMS. The chat LLM has two custom tools, check_availability and
   book_appointment; point their URLs at this clinic's n8n webhooks (same
   paths, same n8n instance = no change if you reuse the instance).
3. Both agents: set the two Retell tool/function URLs (check_availability,
   book_appointment) and the voice agent post-call webhook to the n8n
   production URLs (see Step D). Publish both agents before go-live.
4. Record voice_agent_id, conversation_flow_id, chat_agent_id, llm_id,
   from_number, transfer number.

### Step C. n8n workflows (editor or API)
If reusing the same n8n instance, duplicate the five workflows or keep one set
per clinic (each needs its own webhook paths if multiple clinics share one
instance; see the note at the end). For one clinic per instance, just edit the
existing five. For each workflow, open its "Clinic Config" Set node and set:

- WF-A check-availability > Clinic Config: calendar_id, clinic_tz,
  location_id.
- WF-B book-appointment > Clinic Config: calendar_id, location_id,
  booked_stage_id (= Booked Appointment stage), pipeline_id, clinic_tz.
- WF-C post-call > Clinic Config: location_id, pipeline_id, stage_booked,
  stage_engaged, stage_followup, stage_dnd, stage_no_answer,
  field_call_outcome, field_call_summary, field_retell_call_id.
- WF-D initiate-call > Clinic Config: from_number (the Retell number),
  voice_agent_id, clinic_tz, field_attempt_count.
- WF-E inbound-sms > Clinic Config: chat_agent_id, location_id, field_chat_id
  (= Retell Chat ID custom field).

Then attach credentials:
- Every GHL HTTP node uses the Bearer Auth (GHL PIT) credential.
- The three Retell HTTP nodes (WF-D Create Call, WF-E Create Chat, WF-E Create
  Completion) use the Header Auth (Retell API) credential. Value format is
  header name Authorization, value "Bearer <retell key>". Create this once per
  n8n instance; a human types the key, never an agent, never in workflow text.

Do NOT change the webhook paths or any code node.

### Step D. GHL automation workflows W1-W5 (UI only, not API)
GHL workflows are not in the API. Build them per docs/ghl.md, W1 to W5. Per
clinic you must:
1. W1 trigger: select THIS clinic's lead form.
2. W5 trigger: select THIS clinic's consultation calendar.
3. W2, W3 triggers: Customer Replied, channel SMS.
4. Every Custom Webhook action: point at this n8n instance's production URLs:
   - W1 -> {n8n}/webhook/ghl/initiate-call, body keys contact_id,
     opportunity_id, phone, first_name, last_name, service_of_interest,
     attempt (exact spelling; WF-D reads these).
   - W2 -> {n8n}/webhook/ghl/inbound-sms, body keys contact_id, message,
     first_name, service_of_interest.
5. Business hours: do NOT build per-step time gates. Use the workflow-level
   Time window (Settings > Communication > Specific time), set to 9:00 AM to
   9:00 PM in the clinic timezone, on the desired days. This gates every
   action including the webhook to n8n. Set it on W1 and W4 at least. See the
   dedicated section in docs/ghl.md.
6. W1 pre-call SMS: write out the clinic's Retell number in the message text.
7. W5 confirmation SMS/email: set the clinic name and address.
8. W4 nurture: set the six SMS variants to the clinic's voice.
9. Enable native STOP keyword compliance in phone settings (W2 note).

### Step E. Wire the numbers and publish
1. Buy/import the clinic's Retell phone number and bind the voice agent to it
   as an outbound agent.
2. Confirm the voice agent post-call webhook points at
   {n8n}/webhook/retell/post-call.
3. Publish both Retell agents.
4. A human reviews all five n8n workflows in the editor, then activates them.
   Do not auto-activate.

--------------------------------------------------------------------------
## 3. The per-clinic values worksheet

Fill this in for each new clinic and keep it with the clinic's records.

TIMEZONE (IANA, e.g. America/Chicago): ____________________
  Appears in: GHL account, GHL W1/W4 time windows, all five n8n Clinic Config
  nodes (clinic_tz), both Retell agent timezones.

PHONE NUMBERS
  Retell outbound number (E.164): ____________________
    Appears in: WF-D Clinic Config from_number, the Retell number binding,
    and the W1 pre-call SMS text.
  Office transfer number (E.164): ____________________
    Appears in: the voice conversation flow Transfer node.

GHL IDs
  location_id: ____________________
  pipeline_id: ____________________
  stage_new_lead: ____________________
  stage_engaged (Contacted): ____________________
  stage_booked: ____________________
  stage_followup: ____________________
  stage_no_answer: ____________________
  stage_dnd: ____________________
  field_call_outcome: ____________________
  field_call_summary: ____________________
  field_retell_call_id: ____________________
  field_attempt_count: ____________________
  field_service_of_interest: ____________________
  field_chat_id (Retell Chat ID): ____________________
  calendar_id: ____________________
  lead form: ____________________

RETELL IDs
  voice_agent_id: ____________________
  conversation_flow_id: ____________________
  chat_agent_id: ____________________
  chat llm_id: ____________________

CREDENTIALS (n8n only, never written in workflows or docs)
  GHL Private Integration Token -> n8n Bearer Auth credential
  Retell API key -> n8n Header Auth credential "Retell API"

CLINIC CONTENT (edit in BOTH Retell prompts: voice flow global_prompt and
chat LLM general_prompt)
  Clinic name, address, office hours, services list, current offer, standard
  pricing line, qualification questions, disqualification rules, emergency
  guidance, transfer instruction, required disclaimer.

--------------------------------------------------------------------------
## 4. Verification before go-live

Run the same checks used on the template clinic (all passed July 24, 2026):

1. check-availability: POST the retell/check-availability webhook, expect a
   200 with a "response" string and a non-empty "slots" array pulled from the
   real calendar.
2. inbound-sms plumbing + booking: POST retell/inbound-sms as a test lead
   asking for times, then a second message picking one. Expect the chat agent
   to call check_availability then book_appointment, and a real appointment to
   appear on the calendar. Confirm the Send SMS node returns a messageId (a
   green execution alone does NOT mean the SMS sent).
3. book-appointment direct: POST retell/book-appointment with a live slot,
   expect a 200 spoken confirmation and a calendar event.
4. post-call: POST retell/post-call with a call_analyzed event, expect the
   contact's Call Outcome, Call Summary, Retell Call ID fields updated and a
   note added. Test with a summary containing a quote to confirm valid JSON.
5. initiate-call (COSTS MONEY, a real call): only against a phone you control,
   after publishing the voice agent. Expect a call_id in the Create Call node
   output.

Use only test contacts tagged test-lead. Delete test appointments afterward.

--------------------------------------------------------------------------
## 5. Multiple clinics on one n8n instance

If you run more than one clinic on a single n8n instance, the five webhook
paths would collide (each path can belong to only one workflow). Options:
- One n8n instance per clinic (simplest, matches the template).
- Or namespace the paths per clinic, e.g. retell/clinicB/check-availability,
  and update every Retell tool URL and GHL webhook action to match. If you do
  this, the paths are still contracts: change them in n8n AND in Retell AND in
  GHL together, or the integration breaks.

The current template uses one instance for one clinic.
