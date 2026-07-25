# Manual steps: what an agent cannot do, and exactly how to do it

Written July 24, 2026 after the live audit. Everything an agent could fix has
been fixed (see docs/status.md). What remains needs a human, either because
the API does not expose it, because it handles a secret, or because it is a
go-live action.

Order matters: 1 and 2 unblock the first live call. 3 and 4 unblock WF-E.

---

## 1. Attach the Retell credential to three n8n nodes

**Why an agent cannot do it:** the n8n public API returns HTTP 405 on
`GET /credentials`, so there is no way to look up the existing credential's
ID and link it to a node. Creating a new one instead would mean handling the
Retell API key in plain text, which the no-secrets rule forbids.

**Symptom if skipped:** every outbound call and every SMS chat reply fails on
authentication. WF-D looks like it ran, but Retell never places the call.

**The three nodes:**

| Workflow | Editor URL | Node |
|---|---|---|
| WF-D initiate-call | `https://YOUR_N8N_HOST/workflow/YOUR_WF_INITIATE_CALL_ID` | Create Call |
| WF-E inbound-sms | `https://YOUR_N8N_HOST/workflow/YOUR_WF_INBOUND_SMS_ID` | Create Chat |
| WF-E inbound-sms | same as above | Create Completion |

**IMPORTANT, verified July 24, 2026:** the Retell credential DOES NOT EXIST.
The n8n instance has only two credentials, "Bearer Auth account 2" (Bearer
Auth, used by every GHL node) and "n8n free OpenAI API credits". So this is
not a case of picking an existing entry from a dropdown; the credential has
to be created once, and only you can do it because it means typing the live
Retell API key.

**Step A, create the credential once:**

1. Open https://YOUR_N8N_HOST/home/credentials
2. Click **Create credential**, search for and choose **Header Auth**.
3. Fill in:
   - Name: `Retell API`
   - Header Name: `Authorization`
   - Header Value: `Bearer YOUR_RETELL_API_KEY`
   That is the word Bearer, one space, then the key. The key never goes into
   workflow JSON, this repo, or any chat message.
4. Save.

**Step B, attach it to each of the three nodes:**

1. Open the editor URL for the workflow.
2. Double-click the node.
3. It already has **Authentication = Generic Credential Type** and **Generic
   Auth Type = Header Auth**. Do not change these.
4. In **Header Auth**, open the dropdown and pick **Retell API**. After
   step A it will now appear in the list.
5. Close the node, then **Save** the workflow.

Once step A is done, tell me and I can verify all three nodes are correctly
linked via the API, and run an end-to-end test.

Do all three before testing. WF-E has two nodes on the same canvas, easy to
attach one and forget the other.

**How to verify:** open the node and confirm the credential name shows in the
dropdown rather than a red "Credential not set" warning. A live check is only
possible after step 3 and 4 below.

---

## 2. Publish the Retell agent

**Why an agent cannot do it:** `publishAgentVersion` is a go-live action and
was blocked by the local permission classifier.

**Symptom if skipped:** the agent and its conversation flow both report
`is_published: false`. The phone number binds `agent_version: 1`. Behaviour on
an unpublished version is not guaranteed to match what you see in the canvas.

**Steps:**

1. Open the Retell dashboard, go to Agents.
2. Select **Conversation Flow Agent**
   (`YOUR_VOICE_AGENT_ID`).
3. Before publishing, confirm the two fixes applied via API on July 24 are
   visible in the canvas:
   - Agent timezone reads **America/Chicago**, not America/New_York.
   - The **Goodbye - Wrong Number** node has an arrow leading to **End Call**.
     Previously it dead-ended.
4. Click **Publish**.
5. Confirm the version indicator no longer says Draft.

The phone number YOUR_RETELL_NUMBER is already bound to this agent as
`outbound_agents` weight 1, so nothing to change there.

---

## 3. GHL custom field `retell_chat_id`: DONE

Created July 24, 2026 via the browser on instruction. ID
`YOUR_FIELD_CHAT_ID`, type TEXT, key `contact.retell_chat_id`, Contact
folder. Already wired into WF-E Clinic Config as `field_chat_id`, so SMS
conversations now keep their history between messages.

The original instructions are kept below for the next clinic clone.

### (Reference for future clinics) Create the GHL custom field `retell_chat_id`

**Why an agent cannot do it:** the GHL MCP exposes only
`locations_get-custom-fields`, with no create equivalent. docs/ghl.md also
declares custom fields hand-managed, since the IDs in docs/ids.md are treated
as contracts.

**Symptom if skipped:** WF-E cannot persist a chat ID, so every inbound SMS
starts a brand new Retell chat with no memory of the previous message.

**Steps:**

1. GHL, sub-account **PM Developer 2**
   (`YOUR_LOCATION_ID`).
2. Settings, then **Custom Fields**.
3. **Add Field**, object **Contact**.
4. Field type: **Single Line Text**.
5. Name it `Retell Chat ID`. Confirm the generated field key is
   `contact.retell_chat_id`. If GHL generates something else, edit the key so
   it matches, since ids.md documents that name.
6. Placeholder, optional: `Retell chat session ID, set by n8n`.
7. Save.

**Then get its ID.** The ID is the 20-character string, not the field key.
Two ways:

- Open the field in the UI and read the ID out of the browser URL, or
- ask me to run `locations_get-custom-fields` and read it back to you. That
  is a read-only call and is not blocked.

**Then record it in two places:**

- `docs/ids.md`, replacing the `field_chat_id: NOT CREATED YET` line.
- WF-E `Clinic Config` node, the `field_chat_id` assignment, currently
  `FILL_ME`.

I can do both edits once you give me the ID.

---

## 4. Retell chat agent: DONE, but review the prompt

Created July 24, 2026 on instruction, as a test-grade build.

- Chat agent: `YOUR_CHAT_AGENT_ID`, "Maya SMS Chat Agent"
- Response engine: `YOUR_CHAT_LLM_ID`, gpt-5.1
- `chat_agent_id` is already set in WF-E Clinic Config

The prompt was adapted from the voice global prompt with SMS-specific
constraints: hard 320-character cap, one question per message maximum, no
markdown or emoji, no sign-off, and an explicit rule that this channel has NO
calendar access so it can never state or agree to an appointment time. It
defers all booking to the front desk instead.

Smoke-tested live through the same two API calls WF-E makes. Input: "how much
is it? and my lower back has been hurting for like 3 months". Reply was 224
characters, gave the correct $49 offer, one line of empathy, and offered a
front-desk callback without inventing a time.

**Still to do here:** this is a first draft, not reviewed copy. Read it in the
dashboard before any real lead sees it, especially the pricing lines and the
emergency guidance. The agent is unpublished (version 0), same as the voice
agent, so it is covered by the publish step in section 2.

**Known limitation until section 3 is done:** with `field_chat_id` still
FILL_ME, WF-E's Read Chat ID node never finds a stored chat, so every inbound
SMS starts a fresh Retell chat with no memory of the previous message. It
replies correctly but has no conversation history. Creating the GHL field
fixes this.

---

## 5. Build GHL Workflows W1 to W5

**Why an agent cannot do it:** GHL automation workflows are not exposed in the
API or the MCP at all. docs/ghl.md is explicit that they are UI-only and that
an agent must never claim to have created one.

The full node-by-node spec is already in **docs/ghl.md**, sections W1 to W5.
Build them there. Two things to be careful about:

- **W1 step 9**, the Custom Webhook action, must POST exactly these keys, or
  WF-D's Validate Phone code reads undefined: `contact_id`, `opportunity_id`,
  `phone`, `first_name`, `last_name`, `service_of_interest`, `attempt`.
  Target URL: `https://YOUR_N8N_HOST/webhook/ghl/initiate-call`
- **W2 step 2** posts to
  `https://YOUR_N8N_HOST/webhook/ghl/inbound-sms` with
  `contact_id`, `message`, `first_name`, `service_of_interest`.

Also enable native STOP keyword compliance in phone settings, per W2.

---

## 6. Test WF-B and WF-C

**Why an agent could not do it:** both probe attempts were blocked by the
local permission classifier, once via curl and once via n8n's own test tool.
The two WF-B `onError` fixes from the audit are therefore reasoned from the
graph, not observed running.

You can run these yourself. In this session, prefix with `!` to run the
command and drop the output straight into our conversation.

**6a. WF-B failure path.** Uses a nonexistent contact, so nothing is written
and no appointment is created. Expect HTTP 200 within a couple of seconds,
with a `response` string. If it instead hangs for 120 seconds, the
Move Opportunity fix did not take.

```
curl -s -m 130 -X POST \
  'https://YOUR_N8N_HOST/webhook/retell/book-appointment' \
  -H 'Content-Type: application/json' \
  -d '{"call":{"call_id":"probe"},"args":{"contact_id":"NONEXISTENT_PROBE_ID","slot_start":"2026-08-01T10:00:00-05:00"}}' \
  -w '\nHTTP %{http_code}\n'
```

**6b. WF-B success path.** This one books a real appointment, so use a
throwaway contact. Ask me to create a contact named `Test Booking` tagged
`test-lead`, which docs/ghl.md permits, and I will give you its ID. Then
substitute that ID and a `slot_start` taken verbatim from a live
check-availability response.

**6c. WF-C post-call.** Substitute the same test contact ID. Expect HTTP 200
immediately, since this webhook responds on receive rather than waiting.

```
curl -s -X POST \
  'https://YOUR_N8N_HOST/webhook/retell/post-call' \
  -H 'Content-Type: application/json' \
  -d '{"event":"call_analyzed","call":{"call_id":"probe_wfc","disconnection_reason":"user_hangup","metadata":{"contact_id":"YOUR_TEST_CONTACT_ID","opportunity_id":"YOUR_TEST_OPP_ID"},"call_analysis":{"custom_analysis_data":{"call_outcome":"follow_up","call_summary":"Lead said \"call me next week\", she is busy.","booked_time":""}}}}' \
  -w '\nHTTP %{http_code}\n'
```

That summary deliberately contains a double quote. Before the audit fix it
would have produced invalid JSON and a 400 from GHL. It should now succeed.

**After any of these**, ask me to fetch the test contact. I can read back its
tags, custom field values, opportunity stage, and notes to confirm the
workflow actually wrote what it should. That is the verification pattern in
docs/ghl.md and none of it is blocked.

---

## Quick reference: what is blocked and why

| Item | Blocked by | Agent could ever do it? |
|---|---|---|
| Attach Retell credential | n8n API 405 on credential read, plus secret handling | No |
| Publish Retell agent | Local permission classifier, go-live action | Only with your approval |
| Create GHL custom field | No create tool in GHL MCP, plus ghl.md rule | No |
| Create Retell chat agent | DONE July 24, prompt needs your review | Done |
| GHL Workflows W1-W5 | Not in the GHL API at all | No |
| Test WF-B / WF-C | Local permission classifier | Only with your approval |
