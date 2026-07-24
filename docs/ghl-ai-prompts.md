# GHL AI Workflow Builder prompts (W1-W5)

Paste each block below into GHL's AI workflow builder ("+ Create Workflow"
> "Describe your workflow" / AI Assistant). The AI drafts a workflow; you
then REVIEW every step, because the AI often gets these things wrong and
they must be fixed by hand:

- Webhook action payload keys (must match the n8n contract EXACTLY).
- Custom field / tag / stage names (must match, not near-matches).
- "Wait until business-hours window" configuration.
- The 5 repeated attempt blocks in W1 (AI may collapse or skip them).
- Merge fields: confirm {{opportunity.id}}, {{contact.service_of_interest}},
  and {{message.body}} actually resolve in a test run; rename to whatever
  GHL shows in the merge-field picker if they don't.

Shared facts the AI should use (all workflows):
- Sub-account: Comfort Care Physiotherapy, timezone America/Chicago.
- Pipeline: "SimplicityAI Pipeline". Stages: New Lead, Contacted, Booked
  Appointment, Follow-up, Did Not Answer, DND / Stop.
- Tags: ai-calling-active, booked, dnd-stop, not-interested, nurture-45,
  invalid-phone, transferred-office.
- Contact custom fields: service_of_interest, call_attempt_count.
- n8n base URL: https://YOUR_N8N_HOST
- Retell caller ID (spoken in SMS): +1 828-624-5941

===============================================================================
W1 - Five AI Call Attempts
===============================================================================

Build a GoHighLevel workflow named "W1 - Five AI Call Attempts".

TRIGGER: Form Submitted, for our lead-capture form (the physiotherapy
landing-page form). Set workflow re-entry to OFF.

Then, in order:

1. If/Else condition "Already in progress": if the contact has ANY of the
   tags ai-calling-active, booked, or dnd-stop, stop this workflow
   immediately. Otherwise continue.

2. Create/Update Opportunity: pipeline "SimplicityAI Pipeline", stage
   "New Lead", opportunity name = the contact's full name, status Open.
   Do NOT allow duplicate opportunities for this contact.

3. Add Contact Tag: ai-calling-active.

4. If/Else condition "Has phone": if the contact's Phone field is empty,
   go to a branch that adds the tag invalid-phone, sends an internal
   notification/email to the admin saying "Lead has no phone number, AI
   calling skipped", and stops the workflow. Otherwise continue.

5. Send SMS to the contact:
   "Hi {{contact.first_name}}, this is Comfort Care Physiotherapy. Our
   assistant will call you in the next minute from +1 828-624-5941. Please
   pick up!"

6. Wait 1 minute.

Now repeat the following ATTEMPT BLOCK five times, in sequence (attempts 1
through 5). Each block is identical except for the "attempt" number sent in
the webhook. Create all five explicitly; do not shorten or loop.

   ATTEMPT BLOCK (attempt N = 1,2,3,4,5):
   a. Wait until business hours: wait until the current time in the
      contact's timezone (America/Chicago) is between 9:00 AM and 9:00 PM.
      If it is already within that window, continue with no delay.
   b. If/Else "Stop rule": if the contact has ANY of the tags booked,
      dnd-stop, not-interested, or transferred-office, then remove the tag
      ai-calling-active and stop the workflow. Otherwise continue.
   c. Webhook (POST) to https://YOUR_N8N_HOST/webhook/ghl/initiate-call
      with a JSON body containing these exact keys:
        contact_id         = {{contact.id}}
        opportunity_id      = {{opportunity.id}}
        phone              = {{contact.phone}}
        first_name         = {{contact.first_name}}
        last_name          = {{contact.last_name}}
        service_of_interest = {{contact.service_of_interest}}
        attempt            = N
   d. Wait 4 hours before the next attempt block. (This 4-hour gap is the
      call spacing; it can be tuned later.)

After the fifth attempt block completes without the contact being booked:
7. Remove the tag ai-calling-active.
8. Add the tag nurture-45.
9. Add the contact to the workflow "W4 - 45-Day Nurture".
10. Stop the workflow.

Important: the webhook in each attempt must come AFTER the Create
Opportunity step (step 2) so that {{opportunity.id}} resolves to the
opportunity created here.

===============================================================================
W2 - SMS Chat Bridge
===============================================================================

Build a GoHighLevel workflow named "W2 - SMS Chat Bridge".

TRIGGER: Customer Replied, channel = SMS (fires when a contact sends us an
inbound text message).

Then, in order:

1. If/Else "Opted out": if the contact has the tag dnd-stop, stop the
   workflow immediately. Otherwise continue.

2. Webhook (POST) to https://YOUR_N8N_HOST/webhook/ghl/inbound-sms
   with a JSON body containing these exact keys:
      contact_id         = {{contact.id}}
      message            = {{message.body}}
      first_name         = {{contact.first_name}}
      service_of_interest = {{contact.service_of_interest}}

3. Stop the workflow. (The reply text is sent back to the contact by n8n,
   not by this workflow, so do NOT add any Send SMS step here.)

Also make sure native STOP-keyword compliance is enabled in the phone/SMS
settings for this sub-account (this is a phone-number setting, not part of
this workflow).

===============================================================================
W3 - Pipeline Movement
===============================================================================

Build a GoHighLevel workflow named "W3 - Pipeline Movement".

TRIGGER: Customer Replied, channel = SMS.

Then, in order:

1. If/Else "Already booked": if the contact's opportunity in the
   "SimplicityAI Pipeline" is in stage "Booked Appointment", stop the
   workflow immediately (never move a booked lead backward).

2. If/Else "Engage": if the opportunity is in stage "New Lead" AND the
   contact does NOT have the tag booked and does NOT have the tag dnd-stop,
   then Update Opportunity: move it to stage "Contacted". Otherwise do
   nothing and stop.

Keep this workflow lightweight; it only nudges the pipeline stage when a
lead first replies.

===============================================================================
W4 - 45-Day Nurture
===============================================================================

Build a GoHighLevel workflow named "W4 - 45-Day Nurture".

TRIGGER: Contact Tag Added, tag = nurture-45. Set workflow re-entry to OFF.

This workflow sends six nurture texts, one per cycle, spaced about a week
apart, over roughly 45 days. Build six cycles in sequence. Each cycle has
the same shape; only the SMS wording changes per cycle.

   CYCLE (repeat 6 times, messages 1 through 6):
   a. If/Else "Exit check": if the contact has ANY of the tags booked,
      dnd-stop, or not-interested, then remove the tag nurture-45 and stop
      the workflow. Otherwise continue.
   b. Wait until business hours: wait until the current time in the
      contact's timezone (America/Chicago) is between 9:00 AM and 9:00 PM.
   c. Send SMS to the contact using the wording for this cycle (see the six
      variants below).
   d. Wait 7 days before the next cycle.

Timing: the first message should go out about 3 days after the contact
enters this workflow (you can add a single "Wait 3 days" step before
cycle 1), then weekly after that.

Use these six distinct SMS variants (offer-focused, conversational, no two
worded the same). Keep the $49 new-patient assessment offer as the hook:

   1. "Hi {{contact.first_name}}, it's Comfort Care Physiotherapy. Our $49
      new-patient assessment (normally $120) is still open if you'd like to
      get that pain looked at. Want me to find you a time?"
   2. "Hi {{contact.first_name}}, just checking in. A lot of our patients
      wish they'd come in sooner. The $49 assessment includes a full
      consult and a first treatment plan. Interested?"
   3. "{{contact.first_name}}, quick one - are you still dealing with that
      issue you reached out about? We'd love to help. The $49 assessment
      offer is still available this week."
   4. "Hi {{contact.first_name}}, Comfort Care here. No pressure at all -
      just wanted you to know the $49 new-patient assessment is still on
      the table whenever you're ready. Reply and I'll get you booked."
   5. "{{contact.first_name}}, small aches have a way of becoming big ones.
      Our $49 assessment is a low-cost way to get ahead of it. Want a
      couple of times to choose from?"
   6. "Last check-in, {{contact.first_name}} - the $49 new-patient
      assessment offer is still yours if you want it. Just reply YES and
      I'll find you a time. Either way, take care!"

After cycle 6 completes, remove the tag nurture-45 and stop the workflow.

(Future upgrade, not now: replace the static SMS in step c with a webhook
to n8n that returns an AI-generated message. For v1, keep the six static
variants above.)

===============================================================================
W5 - Confirmation and Reminders
===============================================================================

Build a GoHighLevel workflow named "W5 - Confirmation and Reminders".

TRIGGER: Appointment Booked / Customer Booked Appointment, on the
"Consultation" calendar.

Then, in order:

1. Remove the contact from the workflows "W1 - Five AI Call Attempts" and
   "W4 - 45-Day Nurture" (Remove From Workflow action) so we stop calling
   and nurturing once they book.

2. Send SMS confirmation to the contact:
   "You're booked, {{contact.first_name}}! Your appointment at Comfort Care
   Physiotherapy is on {{appointment.start_time}} at 4212 Main Street,
   Suite 120, Frisco, TX. Need to reschedule or cancel? Just reply to this
   text and we'll help."

3. Send Email confirmation to the contact with the same details: clinic
   name Comfort Care Physiotherapy, the appointment date/time
   {{appointment.start_time}}, the address 4212 Main Street, Suite 120,
   Frisco, TX, and a line on how to reschedule or cancel (reply to the
   confirmation text or call the clinic).

4. Wait until 24 hours before the appointment start time, then send a
   reminder SMS:
   "Reminder from Comfort Care Physiotherapy: your appointment is tomorrow,
   {{appointment.start_time}}, at 4212 Main Street, Suite 120, Frisco, TX.
   See you then! Reply here if anything changes."

5. Wait until 2 hours before the appointment start time, then send a
   reminder SMS:
   "See you soon, {{contact.first_name}}! Your Comfort Care Physiotherapy
   appointment is in about 2 hours at 4212 Main Street, Suite 120, Frisco,
   TX."

The 24-hour and 2-hour offsets are the editable reminder timings.
