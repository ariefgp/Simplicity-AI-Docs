# n8n workflow exports (templates)

Importable JSON for the five n8n workflows behind the Simplicity AI clinic
lead follow-up system. These are sanitized templates: every clinic-specific ID,
phone number, and credential has been replaced with a `YOUR_...` /
`REPLACE_WITH_YOUR_...` placeholder. They contain no secrets. Import them into
your own n8n instance and fill in the placeholders.

## Files

| File | Webhook path | Purpose |
|---|---|---|
| 01-check-availability.json | retell/check-availability | Reads open calendar slots for the AI agents |
| 02-book-appointment.json | retell/book-appointment | Books the appointment, tags, moves the opportunity |
| 03-post-call.json | retell/post-call | Processes the finished call, updates outcome/stage/notes |
| 04-initiate-call.json | ghl/initiate-call | Places the outbound AI call via Retell |
| 05-inbound-sms.json | ghl/inbound-sms | Bridges inbound SMS to the Retell chat agent (can book) |

## How to import

1. In n8n: Workflows > top-right menu > Import from File, and pick a JSON file.
   Do this for each of the five.
2. Each imported workflow will show "credential not set" on its HTTP nodes.
   Create the two credentials once and select them on the nodes:
   - A Bearer Auth credential holding your GoHighLevel Private Integration
     Token. Used by every node named with a GHL URL.
   - A Header Auth credential named e.g. "Retell", header `Authorization`,
     value `Bearer <your Retell API key>`. Used by the three Retell nodes
     (Create Call, Create Chat, Create Completion).
3. Open each workflow's "Clinic Config" Set node and replace every `YOUR_...`
   placeholder with your clinic's real values. See the placeholder key below.
4. Point your Retell agent functions and GoHighLevel webhook actions at your
   own n8n instance's webhook URLs (the paths above are fixed contracts and
   should not be renamed).
5. Review, then activate.

## Placeholders to fill in

- YOUR_LOCATION_ID, YOUR_PIPELINE_ID, YOUR_CALENDAR_ID
- YOUR_STAGE_BOOKED, YOUR_STAGE_ENGAGED, YOUR_STAGE_FOLLOWUP, YOUR_STAGE_DND,
  YOUR_STAGE_NO_ANSWER, YOUR_STAGE_NEW_LEAD
- YOUR_FIELD_CALL_OUTCOME, YOUR_FIELD_CALL_SUMMARY, YOUR_FIELD_RETELL_CALL_ID,
  YOUR_FIELD_ATTEMPT_COUNT, YOUR_FIELD_SERVICE, YOUR_FIELD_CHAT_ID
- YOUR_VOICE_AGENT_ID, YOUR_CHAT_AGENT_ID
- YOUR_RETELL_NUMBER (the Retell outbound number, E.164)
- REPLACE_WITH_YOUR_GHL_CREDENTIAL, REPLACE_WITH_YOUR_RETELL_CREDENTIAL
  (these are credential references; selecting your credential in the node UI
  replaces them automatically)
- clinic_tz is left as `America/Chicago` as an example default; change it to
  your clinic timezone.

Full setup guidance is in `../docs/clone-guide.md`. What each workflow does in
plain language is in `../docs/user-guide.md`.
