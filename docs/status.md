# Status as of July 24, 2026

## Audit: July 24, 2026 (live check of GHL, n8n, Retell)

All three MCP servers connect and authenticate. GHL location, n8n instance
and Retell account all verified live. Findings and fixes below.

### Fixed in this pass
- WF-A check-availability was BROKEN in production, not working as the
  previous status claimed. A live POST to the production webhook returned
  HTTP 200 with an empty body; execution 59782 failed with "Referenced node
  doesn't exist". Four separate bugs, all now fixed (the never-modify rule
  was overridden by explicit user decision):
  1. Set node was named "Edit Fields" but the Code node called
     $('Clinic Config'). Renamed the node to "Clinic Config".
  2. Second Code node called $('Prepare Range'), which did not exist.
     Renamed "Code in JavaScript" to "Prepare Range" and
     "Code in JavaScript1" to "Format Slots".
  3. clinic_tz was "America/New York", not a valid IANA name, and the wrong
     zone. Now America/Chicago.
  4. The free-slots URL was missing the leading "=" so {{ $json.calendarId }}
     was sent literally instead of being evaluated. Fixed, and
     "HTTP Request" renamed to "Get Free Slots".
  Re-probed after the fix: returns five real slots with -05:00 offsets.
- WF-D Clinic Config placeholders filled: from_number YOUR_RETELL_NUMBER,
  voice_agent_id agent_027b4f12a2b6520c3e450fb6c3. Verified live on the
  published version.
- WF-B "Add Booked Tag" had no onError, so a tag failure aborted the whole
  workflow and Retell got no response even though the appointment had been
  created. Now continueRegularOutput.
- WF-B "Move Opportunity" was set to continueErrorOutput but its error output
  was not connected to anything, so a failure (for example an empty
  opportunity_id) left the webhook hanging until Retell's 120s timeout. Now
  continueRegularOutput.
- WF-C "Update Contact Fields" and "Add Note" interpolated the call summary
  straight into a JSON string body. A quote or newline in a GPT-written
  summary would have produced invalid JSON and a 400 from GHL. Both now build
  the body with JSON.stringify.
- Retell voice agent timezone was America/New_York. Now America/Chicago.
- Conversation flow node "Goodbye - Wrong Number" had an edge with no
  destination, so that path dead-ended instead of reaching End Call like every
  other goodbye node. Now wired to node-1784695202531. The update required
  resending the full nodes array; global_prompt, both tools, and model_choice
  were verified intact afterwards.
- ids.md calendar_id reconciled to cleS2YdMFlgsDHnZpch4 (verified live).

### GHL Workflows W1-W5: audited and repaired in the browser

All five now exist (built by the user), and all five were failing. 17 errors
total, fixed via browser automation on explicit user instruction, overriding
the "never edit a GHL workflow" rule in docs/ghl.md. All five remain in DRAFT.
Nothing was published.

Nothing had ever run: 0 contacts, 0 opportunities, 0 GHL-originated n8n
executions. These were build-time validation errors, not runtime failures.

- **W2 SMS Reply Webhook Bridge** (2 errors, now 0). Both webhook actions had
  an empty URL, which is the direct reason no GHL traffic ever reached n8n.
  Configured the one on the correct branch to POST
  {n8n}/webhook/ghl/inbound-sms with contact_id, message, first_name,
  service_of_interest. Its payload had been contact.id / name / email / phone,
  which WF-E's Extract node cannot read. Also DELETED a second Custom Webhook
  that sat on the "Contact has dnd-stop Tag" branch: it would have POSTed
  people who texted STOP into n8n and a Retell chat, contradicting the W2 spec
  and billing as a premium per-execution action. That branch now ends, per spec.
- **W5 Appointment Confirmation** (3 errors, now 0). Trigger held a stale
  calendar ID; reselected Consultation (cleS2YdMFlgsDHnZpch4, the only
  calendar in the account and the same one WF-A/WF-B use). Both "Remove from
  Workflow" actions were set to "Current workflow", so they would have removed
  the contact from W5 itself rather than from W1 and W4. Pointed at the right
  ones.
- **W1 Five AI Call Attempts** (6 errors, now 0). Trigger referenced a form
  named "physiotherapy landing page lead-capture form" that does not exist;
  the account has exactly one form, "SimplifyAI Clinic Contact". Reselected it.
  Deleted the 5 broken "Business Hours Gating" steps (see below).
- **W4 45-Day Nurture** (6 errors, now 0). Deleted the 6 broken "Wait for Time
  Window" steps. All six "Wait 7 Days" steps and all six nurture SMS remain.
- **W3 Pipeline Stage Nudge**: 0 errors, untouched.

**The business-hours finding.** 11 of the 17 errors were per-step time gating
built with wait types that cannot express a time of day. The fix was not to
repair them but to delete them: GHL already has a workflow-level Time window
(Settings > Communication) with exactly the right semantics, and it was
already enabled on W1 and W4 at 08:00-17:00 Mon-Fri. Corrected both to
09:00 AM - 9:00 PM to match the spec. It gates every action including the
Custom Webhook to n8n. Details and the reasoning are in docs/ghl.md.

Consequence: the planned change to WF-D (turn its out-of-window branch into a
Wait) is NOT needed. GHL now holds the contact until the window opens and only
then calls WF-D, so WF-D's existing hour>=9 && hour<21 check is a harmless
backstop. WF-D was not modified.

### WF-E inbound-sms: PASSED end-to-end, July 24, 2026

Tested against a real GHL contact tagged test-lead
(Test SmsBridge, 0KWCKUkeiEmvKW6qe1iX).

Run 1, new conversation (execution 59836, 7.3s, 15 nodes):
- Create Chat -> chat_0c1322ffeb599ccf7af53118794 on Maya SMS Chat Agent
- Create Completion -> "Right now the new patient assessment is $49, normally
  $120, and it includes the consultation and your first treatment plan."
  122 chars, correct offer from CLINIC CONFIG, no invented appointment time.
- Save Chat ID -> wrote the chat id to custom field 8IdlfLOqwDi7cm1SD4HO,
  confirmed by re-fetching the contact.
- Send SMS -> GHL messageId AqMCkX1NZHVD983Mxs0f.

Run 2, same contact, message "what was that price again?" (execution 59837,
3.9s, 12 nodes):
- Read Chat ID found the stored chat, so IF Has Chat routed straight to
  Create Completion and Create Chat was SKIPPED. 12 nodes vs 15 proves the
  branch worked.
- Create Completion returned response_id 2 on the SAME chat, and the agent
  answered the follow-up correctly, which is only possible with prior
  context. Conversation memory is working.
- Send SMS -> messageId BrLhiiQSuw8TXVYfLAUj, same conversationId
  mPXHXOv5MoRuBM3zmj5v as run 1.

Lesson worth keeping: execution 59833 reported status "success" while both
Retell nodes returned 401 and no SMS was sent. The onError:
continueRegularOutput settings make auth failures silent. A green execution
does NOT mean the SMS went out. Always check the Send SMS node output for a
messageId.

### Chat AI can now BOOK by SMS, July 24, 2026 (spec section 6 gap closed)

The chat agent originally had no calendar access and deferred booking to the
front desk, which did not meet spec section 6 ("Booking must use the same n8n
and GoHighLevel calendar functions used by the Voice AI agent"). Fixed by
adding both custom tools to the chat LLM (llm_631bc396538dfb70df2ad9dd8838):
check_availability and book_appointment, pointing at the SAME production
webhooks the voice flow uses, with the same parameter schemas. The BOOKING BY
TEXT section of the prompt was rewritten from "you do NOT have calendar
access" to full booking instructions (offer 2-3 returned slots, only book a
returned slot, confirm only on success).

Tested end to end through WF-E on contact iU4unB90hQ0NtHtGayBr:
- Message 1 "what times do you have?" (execution 59852): agent called
  check_availability, got real slots, replied "I can get you in Friday at
  2:00, 2:30, or 4:00 PM, do any of these work?" Offered 3 of 5, closest
  first.
- Message 2 "2 PM works great" (execution 59854): agent called
  book_appointment with slot_start 2026-07-24T14:00:00-05:00, WF-B returned
  success, agent confirmed "You're all set for your new patient assessment on
  Friday, July 24 at 2:00 PM at Comfort Care Physiotherapy, 4212 Main Street,
  Suite 120, Frisco." Same chat continued (Create Chat skipped).
So SMS can now check availability and book a real appointment, same as voice.

BUG found and fixed during this test: the chat channel has no opportunity_id
dynamic variable, so Retell passed the LITERAL string "{{opportunity_id}}" as
the book_appointment arg. WF-B Extract Args treated that as truthy, which
would bypass the Find Opportunity lookup and PUT to a garbage URL. Fixed
Extract Args to blank out any value containing "{{", so the contact-lookup
fallback runs. Voice path is unaffected (it passes a resolved id with no
braces).

### WF-C post-call: PASSED, July 24, 2026 (execution 59839)

Sent event call_analyzed for contact 0KWCKUkeiEmvKW6qe1iX with a summary
deliberately containing BOTH a double quote and a newline, the exact input
that would have broken the old string-interpolated JSON body.

- Extract -> outcome follow_up, outcomeLabel "Follow-up". Correct mapping.
- Followup Stage -> 404 "Cannot PUT /opportunities/" because the test payload
  had an empty opportunity_id. This is CORRECT behaviour: onError
  continueRegularOutput let the run continue to the contact update instead of
  aborting. Not a defect.
- Update Contact Fields -> wrote call_outcome "Follow-up", retell_call_id
  test_call_wfc_001, and the call summary with the quote and newline intact.
- Add Note -> note rW9KI9MHvHATFHsBWYQa created, body preserved verbatim.

This confirms the JSON.stringify fix. Quote and newline survived end to end.

### WF-B book-appointment: PASSED + hardened, July 24, 2026

First test (execution 59841) booked a real appointment
(R1qnwjkUiMr801UX2OtW, 3:30 PM) and proved the response path: Move Opportunity
hit a 404 on the empty test opportunity_id but the workflow CONTINUED and
returned a 200 in 1.9s instead of hanging. That confirmed the earlier onError
fix.

Then hardened the opportunity step so a missing opportunity_id can no longer
produce a 404 or a stuck pipeline. book_appointment's opportunity_id is
OPTIONAL in the Retell tool schema, so the agent may omit it. New sub-flow
after Add Booked Tag:
  Find Opportunity (GET /opportunities/search by contact_id) ->
  Resolve Opp Id (code: passed opportunityId || first found id) ->
  IF Has Opp -> true: Move Opportunity -> Format Success
              -> false: Format Success (skip the PUT)
Move Opportunity URL now uses the resolved id.

Re-test (execution 59847, empty opportunity_id, contact with no opportunity):
Find Opportunity returned opportunities:[] cleanly, Resolve Opp Id -> "",
IF Has Opp took the false branch, Move Opportunity did NOT run, no 404,
clean 200. Guard confirmed.

Behaviour now: id passed -> used directly; id missing but opportunity exists
-> found by lookup and moved; no opportunity -> skipped, booking still
succeeds. Only the negative path was exercised live (the test contact has no
opportunity and the GHL MCP has no create-opportunity tool), but the search
node ran successfully with the correct response shape.

TEST DATA TO DELETE (all Jul 24, all titled "AI Booked Assessment"):
- contact 0KWCKUkeiEmvKW6qe1iX (Test SmsBridge): appointments at 3:00 PM and
  3:30 PM.
- contact iU4unB90hQ0NtHtGayBr (Test ChatBook): appointment at 2:00 PM, booked
  by the SMS chat agent during the section-6 test. Has the booked tag.
Both contacts are tagged test-lead. The GHL user currently cannot delete
contacts (permission off), so remove the appointments in the calendar and
optionally leave the contacts.

### Verified healthy, no change needed
- GHL: location 1nw8goHWjkKjkpdfUWIQ, pipeline and all 6 stage IDs, 5 of the
  6 custom fields present with IDs matching ids.md, call_outcome picklist
  labels match the WF-C label map exactly.
- Retell: voice agent post-call webhook points at WF-C, post-call extraction
  fields (call_outcome, call_summary, booked_time) present, both flow tools
  point at the production webhook URLs, book_appointment carries
  opportunity_id, phone number YOUR_RETELL_NUMBER binds the voice agent as
  outbound_agents weight 1.
- n8n: all 5 workflows exist and are active. WF-C routing, switch outputs,
  label mapping, and Version headers all correct.

## Outstanding (human action required)

Step-by-step instructions for every item below are in docs/manual-steps.md.

1. DONE July 24, 2026. The user created the "Retell API" Header Auth
   credential (id NUR7UGXDR9tzLKEG) and attached it to all three nodes:
   WF-D "Create Call", WF-E "Create Chat", WF-E "Create Completion".
   Verified on the published versions, not just drafts.
   History: the first attempt returned 401 "Invalid API Key" on both Retell
   nodes (execution 59833); the user re-pasted the key and it now works.
   WF-E IS NOW TESTED END TO END. See the test log below.
2. Retell agent and conversation flow both report is_published: false. The
   phone number binds agent_version 1. Publish in the dashboard before the
   first live call. The publishAgentVersion API call was blocked by the local
   permission classifier as a go-live action.
3. DONE. GHL custom field "Retell Chat ID" created July 24, 2026 via the
   browser on explicit user instruction, overriding the hand-managed rule in
   docs/ghl.md (the GHL MCP has no create-custom-field tool, so the UI was
   the only route). ID 8IdlfLOqwDi7cm1SD4HO, TEXT, key
   contact.retell_chat_id, Contact folder. Verified via the API and wired
   into WF-E Clinic Config as field_chat_id on the published version.
   WF-E now has no FILL_ME values left.
4. DONE. Chat agent created July 24, 2026 on user instruction ("this is just
   for test"): agent_a7739f2c1375fc9b1051456b74, "Maya SMS Chat Agent", on
   retell-llm llm_631bc396538dfb70df2ad9dd8838 (gpt-5.1). SMS prompt adapted
   from the voice global prompt: hard 320-character cap, one question max, no
   markdown or emoji, and explicitly NO calendar access in this channel so it
   can never invent an appointment time. Smoke-tested live via create-chat +
   create-chat-completion; the reply was 224 characters, quoted the correct
   $49 offer, and correctly deferred booking to the front desk. chat_agent_id
   is now set in WF-E Clinic Config. The prompt is first-draft quality and
   should be reviewed before any real lead sees it.
5. GHL Workflows W1-W5 now exist and all show 0 errors, but all five are
   still in DRAFT and have never run. Publish them when ready. Before
   publishing, sanity-check the two things an agent should not decide:
   whether Saturday belongs in the 9:00-21:00 window (currently off), and
   whether the six nurture SMS bodies read the way the clinic wants.
6. WF-A, WF-C and WF-E are now TESTED and passing (see logs above).
   Still untested, because both have real-world side effects and need the
   user's go-ahead:
   - WF-B book-appointment: writes a real appointment into the Consultation
     calendar. Needs deleting afterwards.
   - WF-D initiate-call: DELIBERATELY NOT TESTED. The final step,
     POST /v2/create-phone-call, places a REAL billable call (Retell
     per-minute + Twilio telephony + LLM tokens). User chose July 24, 2026 to
     skip it rather than incur cost. Everything upstream is verified:
     the Retell credential is the SAME one WF-E used successfully, so auth is
     proven; Validate Phone, Check Window, IF Stopped and Clinic Config
     (from_number YOUR_RETELL_NUMBER, voice_agent_id) are inspected and correct.
     Only the actual call placement is unproven, and only because it costs
     money. When ready, point it at a phone the user controls, never a lead,
     keep the call short, and check the Create Call node output for a
     call_id.

## Next (in order)
0. Delete the test contact "Test SmsBridge" (0KWCKUkeiEmvKW6qe1iX, tagged
   test-lead) in the GHL UI. The GHL MCP has no delete-contact tool, so an
   agent cannot remove it. It also holds a real Retell chat id in the
   retell_chat_id field.
1. Test WF-B and WF-C via webhook (item 6).
2. Publish the Retell agent (item 2).
4. Build GHL Workflows W1-W5 in the UI.
5. Create retell_chat_id field + chat agent, then test WF-E.
6. Test WF-D initiate-call, then first live end-to-end call test.
7. Hardening (webhook header auth, signature verification), acceptance test,
   per-clinic setup guide.

## Known debt
- WF-C signature verification was removed earlier (the old node had a
  hardcoded YOUR_RETELL_API_KEY that threw on every call and violated the
  no-secrets-in-code rule). Re-add in Phase 5 using a credential, not code.
- WF-D "Get Contact" has no onError. A 404 aborts the workflow. Acceptable
  for now since WF-D is fire-and-forget with no webhook response, but it
  fails silently from GHL's point of view.

## Decisions log
- WF-B: after appointment creation succeeds, tag/stage errors continue on
  regular output, never fail the response. WF-C repeats stage+tag.
- Outcome label mapping happens in WF-C Extract code.
- Pre-call SMS wording states the Retell number explicitly.
- n8n Variables unavailable: per-workflow "Clinic Config" Set node.
- Stage "Contacted" plays the Engaged/Answered role.
- July 24, 2026: the "never modify WF-A" rule was overridden on explicit user
  instruction after WF-A was proven broken in production.
