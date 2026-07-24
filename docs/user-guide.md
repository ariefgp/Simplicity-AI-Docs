# Simplicity AI: System Documentation and User Guide

A lead follow-up system for clinics. When a lead submits a form, the system
texts them, calls them with an AI voice agent, answers questions by AI SMS,
books real appointments on the clinic calendar, and keeps the CRM pipeline up
to date, all automatically, while respecting opt-outs and calling hours.

This document explains what the system does, how a lead moves through it, what
each part is responsible for, how clinic staff read and act on what it
produces, and how to monitor and troubleshoot it. For cloning the system to a
new clinic, see docs/clone-guide.md. For the raw identifiers, see docs/ids.md.

--------------------------------------------------------------------------
## 1. What the system does, in one paragraph

A new lead fills in the clinic's landing page form. GoHighLevel (the CRM)
stores them and starts the follow-up. Within about a minute they get a text
and an AI phone call from "Maya", who answers questions, qualifies them, and
books an assessment straight into the clinic calendar if they are interested.
If they text instead of answering, a second AI agent handles the SMS
conversation and can also book. Every call and text updates the lead's record
and pipeline stage. If the lead books, all sales follow-up stops and they only
get appointment confirmations and reminders. If they say stop, all contact
ends immediately.

--------------------------------------------------------------------------
## 2. The three systems and what each is responsible for

- GoHighLevel (GHL): the CRM. Holds contacts, the sales pipeline, the
  calendar, SMS sending, the landing page and form, and the five automation
  workflows that decide when to call, text, and stop.
- Retell AI: the AI brains. Two agents. A voice agent handles phone calls. A
  chat agent handles SMS conversations. Both know the clinic's information and
  can check the calendar and book.
- n8n: the connector. Five small workflows that sit between Retell and GHL.
  They place calls, check availability, book appointments, process finished
  calls, and bridge inbound texts to the chat agent.

Nothing talks to the calendar or the CRM directly except through these pieces.
The clinic's information (name, hours, services, prices, offer, rules) lives in
the Retell prompts. The IDs and settings live in the n8n "Clinic Config" nodes.
Secrets (API keys) live only in n8n credentials.

--------------------------------------------------------------------------
## 3. The lead journey, end to end

1. Lead submits the landing page form (name, phone, email, service of
   interest).
2. GHL creates the contact and an opportunity at the "New Lead" stage, and
   tags the lead ai-calling-active.
3. GHL sends a heads-up SMS: the clinic will call in about a minute from the
   AI number, please pick up.
4. GHL calls the n8n initiate-call workflow, which tells Retell to place the
   call. Before calling, n8n checks the number is valid and that the current
   time is inside calling hours.
5. Maya (voice) greets the lead, answers questions using only the clinic's
   information, qualifies them, and if they are interested, checks the
   calendar and offers open times.
6. If the lead picks a time, Maya books it directly into the clinic calendar,
   confirms the details, and offers to transfer them to the front desk.
7. When the call ends, Retell sends the result to n8n, which records the
   outcome, a written summary, and the Retell call ID on the contact, and
   moves the opportunity to the right pipeline stage.
8. If no appointment was booked and the lead did not opt out, GHL waits the
   configured gap and tries again, up to five calls total.
9. If the lead texts back at any point, the AI chat agent answers, can also
   check availability and book, and replies by SMS.
10. If the lead books (by call or text), GHL removes them from all sales
    follow-up and sends confirmation and reminders only.
11. If the lead is never reached after five attempts, they enter a 45-day
    gentle SMS nurture, one message a week.
12. If the lead says stop at any point, all calls and marketing texts end
    immediately.

--------------------------------------------------------------------------
## 4. The five GHL automations (plain language)

These are the workflows in GHL under Automation. All five exist and validate
clean. They are currently in Draft; publish them to go live.

- W1, Five AI Call Attempts. Triggered by a form submission. Tags the lead,
  sends the heads-up SMS, then calls the lead through n8n up to five times with
  a configurable gap between attempts. Skips a lead who has already booked,
  opted out, said not interested, or been transferred.
- W2, SMS Chat Bridge. Triggered when a lead replies by SMS. Sends the message
  to the AI chat agent through n8n and texts back the reply. If the lead texted
  a stop word, it marks them do-not-contact and ends.
- W3, Pipeline Stage Nudge. Triggered by an SMS reply. Moves a New Lead who
  engages into the Contacted stage, unless they have already booked or opted
  out.
- W4, 45-Day Nurture. For leads who never booked. One conversational SMS a
  week for about 45 days, each worded differently. Stops the moment the lead
  books, opts out, or says not interested.
- W5, Appointment Confirmation and Reminders. Triggered when an appointment is
  booked. Removes the lead from W1 and W4, sends a confirmation by SMS and
  email, and sends reminders before the appointment.

Calling and texting hours (9:00 AM to 9:00 PM, clinic time) are enforced by a
single workflow-level Time window in each workflow's settings, not by
individual steps.

--------------------------------------------------------------------------
## 5. The AI agents: what they can and cannot do

Both agents are "Maya" and use only the clinic information in their prompt.
They never invent services, prices, or availability, never give medical
advice, and always identify as AI when asked.

Voice agent (phone calls):
- Answers questions, explains the offer, qualifies the lead.
- Checks the calendar and offers real open times.
- Books the appointment directly. A booking is only real once it is created in
  the calendar; only then does Maya confirm it.
- After booking, transfers the caller to the front desk. If the transfer
  fails, the appointment stays and Maya says the clinic will follow up.
- Leaves a short voicemail if there is no answer.

Chat agent (SMS):
- Same clinic knowledge, tuned for short text replies.
- Answers FAQs and qualifies.
- Checks availability and books, using the same calendar functions as the
  voice agent. Confirms only once the booking succeeds.
- Continues the same conversation across multiple texts (it remembers context
  within a lead's thread).
- If a lead texts a stop word, they are marked do-not-contact upstream before
  the agent replies.

What the agents do NOT do:
- They do not book a time they have not seen from the calendar.
- They do not promise a time if the calendar cannot be reached.
- They do not diagnose or give clinical advice.

--------------------------------------------------------------------------
## 6. User guide for clinic staff

You mostly do not operate this system; it runs itself. Your job is to read what
it produces and handle the handoffs.

### Reading a contact record
Open a contact in GHL. The custom fields tell you the AI's results:
- Call Outcome: the result of the most recent call. One of Appointment Booked,
  Call Transferred to Office, Follow-up, Not Interested, DND/Stop, Did Not
  Answer.
- Call Summary: a short factual summary of what happened on the call.
- Retell Call ID: reference code for looking up the full call in Retell.
- Call Attempt Count: how many times the AI has called.
- Notes: each AI call adds a note with the outcome and summary.

### What the pipeline stages mean
- New Lead: form submitted, not yet engaged.
- Contacted: the lead replied or picked up.
- Booked Appointment: an appointment is on the calendar. This wins over every
  other stage.
- Follow-up: engaged but did not book, still being followed up.
- Did Not Answer: calls got no answer.
- DND / Stop: opted out. No more calls or marketing texts.

### When a call is transferred to you
The voice agent transfers to the front desk after a successful booking, or
when a lead asks for a human. Answer as normal; the appointment is already
booked. If you missed the transfer, the booking still stands.

### When an appointment is booked
The lead is automatically removed from all sales follow-up and moved to
confirmations and reminders only. You do not need to stop anything by hand.

### When a lead opts out
If a lead says stop, remove me, or similar, the system marks them DND/Stop and
ends all calls and marketing texts. Do not re-enroll them.

### Booking through the calendar
All AI bookings land on the clinic's consultation calendar as "AI Booked
Assessment". They appear like any other appointment. Reschedule or cancel them
normally; the lead can also reply to reschedule.

--------------------------------------------------------------------------
## 7. Monitoring and health

- GHL: watch the pipeline board and contact activity. Failed message deliveries
  show a red indicator on the message in the conversation view.
- n8n: open the instance, then each workflow's Executions tab to see runs.
  Green means the workflow finished; it does NOT by itself mean an SMS or call
  went out. To confirm an SMS actually sent, check that the final Send SMS step
  produced a message ID. To confirm a call was placed, check that the Create
  Call step returned a call ID.
- Retell: the dashboard shows call and chat history, transcripts, and cost.
- A quick daily check: any leads stuck in New Lead with attempts climbing but
  no outcome, and any executions marked error in n8n.

--------------------------------------------------------------------------
## 8. Configuration reference (where settings live)

For full detail see docs/clone-guide.md. In short:

- Clinic information (name, address, hours, services, offer, pricing, rules,
  disclaimer): edit in BOTH Retell prompts, the voice conversation flow and the
  chat agent LLM.
- IDs (pipeline, stages, calendar, custom fields, agents, numbers): the
  "Clinic Config" Set node inside each of the five n8n workflows.
- Calling and texting hours: the Time window in each GHL workflow's settings.
- Call spacing (gap between the five attempts): the Wait steps in GHL W1.
- Reminder timing: the Wait-until steps in GHL W5.
- Nurture wording: the six SMS variants in GHL W4.
- Transfer number: the Transfer node in the voice conversation flow.
- API keys: n8n credentials only, never in prompts or workflow text.

--------------------------------------------------------------------------
## 9. Safety, compliance, and error handling

- Opt-out: a stop word marks the lead DND/Stop and ends all calls and
  marketing SMS immediately.
- Calling hours: no calls or texts outside 9:00 AM to 9:00 PM clinic time.
- Invalid number: leads with a bad phone number are tagged and skipped, not
  called.
- Duplicate prevention: duplicate contacts and opportunities are blocked, and
  workflows do not re-enter, so a lead is not called or texted twice for the
  same event.
- AI honesty: Maya identifies as an AI whenever asked and gives emergency
  guidance (contact emergency services) instead of booking if a lead describes
  something urgent.
- Booking safety: nothing is treated as booked until the calendar confirms it.
  If booking fails, the lead is told the clinic will follow up.
- If the calendar cannot be reached, the agent does not promise a time.

--------------------------------------------------------------------------
## 10. Troubleshooting

- SMS did not arrive but the n8n run was green: a green run is not proof of
  delivery. Check the Send SMS step for a message ID, and check the GHL
  conversation for a red delivery-failure marker. A failed delivery usually
  means an invalid or unroutable phone number.
- AI call never happens: check the lead has a valid E.164 phone number, is
  inside calling hours, and does not carry booked, dnd-stop, not-interested, or
  transferred-office tags. Check the n8n initiate-call execution.
- Booking says success but pipeline did not move to Booked: the opportunity ID
  may not have reached the booking step; the system now looks the opportunity
  up by contact as a fallback. Confirm the lead actually has an opportunity in
  the pipeline.
- Chat replies are generic or forget context: confirm the Retell Chat ID field
  on the contact holds a value; that is what lets a conversation continue.
- Calendar shows no open times: confirm the consultation calendar has
  availability configured and the calendar ID in the n8n Clinic Config matches.
- Retell returns Invalid API Key: the Retell credential in n8n needs the value
  in the form "Bearer <key>", with the word Bearer and a space before the key.

--------------------------------------------------------------------------
## 11. Current state (as of July 25, 2026)

Built and verified working end to end:
- check-availability, book-appointment (voice), post-call processing, and the
  SMS chat bridge including SMS booking, all tested against the live calendar
  and CRM.
- Both AI agents exist and are published. All API credentials are attached and
  authenticate.

Built, not yet exercised:
- The outbound call workflow's final call-placement step. Everything up to it
  is verified; the actual call was left untested to avoid the per-call charge.
  Validate it with one short test call to a number you control before go-live.

Pending human actions before go-live:
- Publish the five GHL automations (they are in Draft).
- Confirm the landing page form captures all required fields.
- Run one live test call.

Known limitations:
- Internal failure alerts (for example on a booking failure) are placeholders
  that do not yet notify a person by email or chat.
- The voice agent does not update contact details mid-call (this was optional
  in the specification).

--------------------------------------------------------------------------
## 12. Glossary

- Lead: a person who submitted the form.
- Opportunity: the lead's deal card in the sales pipeline.
- Stage: where the opportunity sits in the pipeline (New Lead, Contacted, etc).
- Outcome: the result the AI recorded for a call.
- DND/Stop: do-not-contact; the lead opted out.
- Nurture: the slow 45-day weekly SMS follow-up for unreached leads.
- Webhook: the URL one system calls to trigger another. These are fixed and
  must not be renamed.
- Clinic Config: the settings node inside each n8n workflow holding this
  clinic's IDs.
