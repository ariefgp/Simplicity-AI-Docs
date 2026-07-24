# GHL via MCP: rules and manual-build spec

## MCP usage rules
- The GHL MCP touches the LIVE client sub-account. Read/verify and test
  data only.
- Allowed: read contacts/opportunities/calendars, create test contacts
  (always tag "test-lead" and name them Test + something), read tags and
  custom fields to verify n8n workflow effects, delete only contacts
  tagged "test-lead".
- Never: bulk updates, deleting or modifying anything not tagged
  test-lead, creating/editing pipelines, calendars, custom fields, or
  tags (those are hand-managed; IDs in docs/ids.md are contracts).
- GHL automation Workflows (W1-W5) are NOT in the API or MCP. They are
  built manually in the GHL UI per the spec below. Never claim to have
  created or edited a GHL workflow.
- Verification pattern after n8n tests: fetch the test contact, report
  tags, custom field values, opportunity stage, and appointments.

## Business hours: use the workflow-level Time window, not per-step waits

Every GHL workflow has Settings > Communication > Time window > Specific time.
Its behaviour is exactly what the 9:00-21:00 rule needs: "Execute actions if
the current time falls in this window, else wait until the next window to
execute the action." It applies to EVERY action in the workflow, including
Custom Webhook actions, so it gates the calls to n8n too.

W1 and W4 are both set to 09:00 AM - 9:00 PM, Monday to Friday, Account
timezone (which resolves to America/Chicago). Set this on any new clinic
workflow instead of adding per-step waits.

Do NOT build business-hours gating as individual Wait steps. The July 24, 2026
build tried that and produced 11 broken steps: "Until specific conditions are
met" only accepts contact-field conditions (name, email, phone), so it can
never express a time of day, and "Until a recurring window opens" always waits
for the NEXT occurrence, which would delay an in-hours lead by up to a day and
break the one-minute callback promise W1's own SMS makes.

Saturday is currently unchecked on both. Turn it on if the clinic wants
weekend outreach.

## Manual build spec: GHL Workflows (UI only)

### W1: Five AI Call Attempts
Trigger: Form Submitted (the lead form).
1. IF tags contain ai-calling-active OR booked OR dnd-stop: End.
2. Create/Update Opportunity: SimplicityAI Pipeline, stage New Lead,
   allow duplicate OFF.
3. Add tag ai-calling-active.
4. IF phone is empty: add tag invalid-phone, internal notification, End.
5. Send SMS: "Hi {{contact.first_name}}, this is Comfort Care
   Physiotherapy. Our assistant will call you in the next minute from
   +1 XXX-XXX-XXXX. Please pick up!" (Retell number written out.)
6. Wait 1 minute.
7. (No step needed. The workflow-level Time window handles 9:00-21:00. See
   the section above.)
8. IF tags contain booked / dnd-stop / not-interested /
   transferred-office: remove ai-calling-active, End.
9. Custom Webhook action: POST to n8n WF-D production URL
   {n8n}/webhook/ghl/initiate-call with JSON body:
   contact_id={{contact.id}}, opportunity_id={{opportunity.id}},
   phone={{contact.phone}}, first_name={{contact.first_name}},
   last_name={{contact.last_name}},
   service_of_interest={{contact.service_of_interest}}, attempt=1
10. Wait 4 hours (THE configurable call gap; document any change).
11. Repeat steps 7-10 as attempts 2,3,4,5 (duplicate the block, bump the
    attempt value each time; GHL has no loops).
12. After attempt 5: remove ai-calling-active, add tag nurture-45,
    Add to Workflow W4, End.

### W2: SMS Chat Bridge
Trigger: Customer Replied, channel SMS.
1. IF tags contain dnd-stop: End.
2. Custom Webhook: POST {n8n}/webhook/ghl/inbound-sms with
   contact_id={{contact.id}}, message={{message.body}},
   first_name={{contact.first_name}},
   service_of_interest={{contact.service_of_interest}}
3. End (reply is sent by n8n).
Also: enable native STOP keyword compliance in phone settings.

### W3: Pipeline Movement
Trigger: Customer Replied (SMS).
1. IF opportunity stage is Booked Appointment: End.
2. IF stage is New Lead AND tags do not contain booked or dnd-stop:
   move opportunity to Contacted (engaged stage).

### W4: 45-Day Nurture
Trigger: added by W1 (or tag nurture-45 added).
Six cycles, each:
1. IF tags contain booked / dnd-stop / not-interested: remove
   nurture-45, End.
2. (No step needed. The workflow-level Time window handles 9:00-21:00, which
   also keeps nurture texts from going out overnight.)
3. Send SMS (v1: six pre-written static variants, one per cycle, offer-
   focused, conversational, different wording each. v2 upgrade: webhook
   to n8n for AI-generated message).
4. Wait 7 days.
Cycle timing: first send ~day 3, then weekly (~45 days total).
Workflow settings: allow re-entry OFF.

### W5: Confirmation and Reminders
Trigger: Appointment Booked (calendar: Consultation).
1. Remove From Workflow: W1, W4.
2. Send SMS + Email confirmation: clinic name, {{appointment.start_time}},
   address 4212 Main Street Suite 120 Frisco TX, reschedule/cancel
   instructions.
3. Wait until 24h before appointment: reminder SMS.
4. Wait until 2h before appointment: reminder SMS.
(24h/2h are the editable offsets.)

## W1 payload contract (must match WF-D's Validate Phone code)
Keys: contact_id, opportunity_id, phone, first_name, last_name,
service_of_interest, attempt. Exact spelling; WF-D reads body.phone,
body.contact_id, etc.
