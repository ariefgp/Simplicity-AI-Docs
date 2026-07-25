# Real identifiers (not secrets)

## GHL
- location_id: YOUR_LOCATION_ID
- pipeline_id: YOUR_PIPELINE_ID (SimplicityAI Pipeline)

### Stage IDs
- stage_new_lead: YOUR_STAGE_NEW_LEAD (New Lead)
- stage_engaged: YOUR_STAGE_ENGAGED (named "Contacted")
- stage_booked: YOUR_STAGE_BOOKED (Booked Appointment)
- stage_followup: YOUR_STAGE_FOLLOWUP (Follow-up)
- stage_no_answer: YOUR_STAGE_NO_ANSWER (Did Not Answer)
- stage_dnd: YOUR_STAGE_DND (DND / Stop)

### Custom field IDs (contact)
- field_call_outcome: YOUR_FIELD_CALL_OUTCOME (SINGLE_OPTIONS, human labels)
- field_call_summary: YOUR_FIELD_CALL_SUMMARY (LARGE_TEXT)
- field_retell_call_id: YOUR_FIELD_RETELL_CALL_ID (TEXT)
- field_attempt_count: YOUR_FIELD_ATTEMPT_COUNT (NUMERICAL)
- field_service_of_interest: YOUR_FIELD_SERVICE (SINGLE_OPTIONS)
- field_chat_id: YOUR_FIELD_CHAT_ID (TEXT, "Retell Chat ID", key
  contact.retell_chat_id, Contact folder). Created July 24, 2026 via the UI
  on user instruction; wired into WF-E Clinic Config.

### call_outcome label map (raw enum -> GHL picklist label)
- appointment_booked -> "Appointment Booked"
- transferred_to_office -> "Call Transferred to Office"
- follow_up -> "Follow-up"
- not_interested -> "Not Interested"
- dnd_stop -> "DND/Stop"
- did_not_answer -> "Did Not Answer"
- unknown -> "Follow-up"

### Tags (exact strings)
ai-calling-active, booked, dnd-stop, not-interested, nurture-45,
invalid-phone, transferred-office

### Calendar
- calendar_id: YOUR_CALENDAR_ID (Consultation calendar; verified live via
  the free-slots endpoint on July 24, 2026)

## Retell
- voice_agent_id: YOUR_VOICE_AGENT_ID (name "Conversation Flow
  Agent", YOUR_CONVERSATION_FLOW_ID)
- chat_agent_id: YOUR_CHAT_AGENT_ID (name "Maya SMS Chat Agent",
  response engine YOUR_CHAT_LLM_ID, gpt-5.1). Created and
  smoke-tested July 24, 2026. Unpublished (version 0), same as the voice agent.
- from_number: YOUR_RETELL_NUMBER (E.164 Retell number, SIP trunk; voice agent
  assigned as outbound_agents weight 1)
- clinic_tz: America/Chicago (agent timezone corrected from America/New_York
  on July 24, 2026)

## n8n
- Instance: https://YOUR_N8N_HOST
- Existing: WF-A check-availability (path retell/check-availability),
  built and tested. Do not modify.
