# Real identifiers (not secrets)

## GHL
- location_id: 1nw8goHWjkKjkpdfUWIQ
- pipeline_id: QqHBuuQtTOYwQ4ZohYqf (SimplicityAI Pipeline)

### Stage IDs
- stage_new_lead: 4fc72a52-16a3-4485-ad93-be53712cfae3 (New Lead)
- stage_engaged: 8c481e2f-c39e-46ae-824d-7a5ecf1e5454 (named "Contacted")
- stage_booked: 84e4ce71-019c-4200-a74c-bc0ebcef6195 (Booked Appointment)
- stage_followup: d9333e44-488e-4346-ac16-ff6679b872fb (Follow-up)
- stage_no_answer: 54e47444-9707-4d29-88c2-06b9628f4719 (Did Not Answer)
- stage_dnd: 3e427463-f338-4eb1-b1a0-237bd90e69e6 (DND / Stop)

### Custom field IDs (contact)
- field_call_outcome: mpZ2JCyulPfXCObSFyve (SINGLE_OPTIONS, human labels)
- field_call_summary: tjD28s5Q2hFhmilNCaWy (LARGE_TEXT)
- field_retell_call_id: UlkYtTeN5WGPplSsvMQv (TEXT)
- field_attempt_count: pPXGL6TTe0Z9Bp3RcfdO (NUMERICAL)
- field_service_of_interest: H3lCHgf3r1Ed5Q0eeaOc (SINGLE_OPTIONS)
- field_chat_id: 8IdlfLOqwDi7cm1SD4HO (TEXT, "Retell Chat ID", key
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
- calendar_id: cleS2YdMFlgsDHnZpch4 (Consultation calendar; verified live via
  the free-slots endpoint on July 24, 2026)

## Retell
- voice_agent_id: agent_027b4f12a2b6520c3e450fb6c3 (name "Conversation Flow
  Agent", conversation_flow_a76cb9158559)
- chat_agent_id: agent_a7739f2c1375fc9b1051456b74 (name "Maya SMS Chat Agent",
  response engine llm_631bc396538dfb70df2ad9dd8838, gpt-5.1). Created and
  smoke-tested July 24, 2026. Unpublished (version 0), same as the voice agent.
- from_number: YOUR_RETELL_NUMBER (E.164 Retell number, SIP trunk; voice agent
  assigned as outbound_agents weight 1)
- clinic_tz: America/Chicago (agent timezone corrected from America/New_York
  on July 24, 2026)

## n8n
- Instance: https://YOUR_N8N_HOST
- Existing: WF-A check-availability (path retell/check-availability),
  built and tested. Do not modify.
