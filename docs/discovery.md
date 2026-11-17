# 1. Discovery

## Selected Domain

**Lead qualification.**

A potential client of a small automation agency writes to a Telegram bot in free
text. The assistant extracts the requested service, company type, budget,
timeframe and contact details, asks a follow-up question when essential
information is missing, classifies the lead against a documented rule, and
records complete leads in Google Sheets. The reply is sent back to the same
Telegram chat, and each chat keeps a separate conversation.

## Acceptance Criteria

| ID | Behaviour |
|---|---|
| AC01 | A free-text Telegram message produces a reply in the same chat |
| AC02 | Two Telegram chats do not share conversation history |
| AC03 | Within one chat, information from earlier messages is retained |
| AC04 | Requested service, company type, budget, timeframe and contact are extracted when present |
| AC05 | Values absent from the message are left empty; no budget, timeframe, company fact or contact is invented |
| AC06 | Approximate or vague values are recorded and marked in the `uncertainty` column |
| AC07 | Every lead is classified `hot`, `warm`, `cold` or `incomplete` according to `qualification_rules` |
| AC08 | The services list is read from Google Sheets |
| AC09 | A service not present in the services list is classified `cold` |
| AC10 | A complete lead is written to the `leads` sheet by an Agent tool call |
| AC11 | An `incomplete` lead is not written; one follow-up question is asked instead |
| AC12 | The execution trace shows the tool call with its input and output |
| AC13 | A request outside the agency's scope receives a safe answer, not an invented one |
| AC14 | Contradictory input is not resolved silently: the conflict is queried or marked |

## Assumptions

| # | Not specified | Assumption |
|---|---|---|
| A1 | Currency of budgets | EUR (`budget_eur`) |
| A2 | Whether timeframe affects qualification | It does not; timeframe is stored but not classified on |
| A3 | Definition of a valid contact | Any email or phone number supplied by the user |
| A4 | Multiple leads from one Telegram user | Allowed; each completed conversation produces one row |
| A5 | Who sets `status` | The assistant always writes `new` |
| A6 | Language of incoming messages | English |
| A7 | Whether cold leads are recorded | Yes, with `cold` and a decline action |

## Research Log

> Rows marked ✅ were verified in practice. Remaining rows to be completed.

| Source | Question | Useful finding | Decision or change | Verification |
|---|---|---|---|---|
| n8n docs — AI Agent node | How does an Agent receive tools? | Tools attach to the Agent's Tool slot as sub-nodes | A Sheets node placed after the Agent would not satisfy the requirement | ✅ Calculator attached to the Tool slot appears in the execution trace |
| Google AI Studio | Which Gemini model is available to a new API key? | `gemini-2.5-flash` returns 404 for new users; `gemini-3.6-flash` works | Model switched | ✅ Tests A and B pass |
| Own testing | Does Gemini perform tool calling from n8n? | Trace shows `Agent → model → Calculator → model` | No provider change needed | ✅ Input `8347 * 2913`, output `24314811` |
| Docker / colima | Docker Desktop not available on the host | colima provides a lightweight Docker engine | Runtime changed | ✅ `docker info`, `curl /healthz` |
| n8n docs — Call n8n Workflow Tool | | | | |
| n8n docs — Google Sheets node | | | | |
| n8nworkflows.xyz — example 1 | | | | |
| n8nworkflows.xyz — example 2 | | | | |

---

# 2. Architecture

## Option A — Google Sheets Tool attached directly to the Agent

The Google Sheets tool node sits in the Agent's Tool slot. The model fills its
parameters directly.

**Trade-offs:** fewest components and one credential. No place for validation,
normalisation or defaults — whatever the model produces is written. Reuse from a
second point requires duplicating the node.

## Option B — Call n8n Workflow Tool with a sub-workflow

The Agent's Tool slot holds a `Call n8n Workflow Tool` that triggers a separate
sub-workflow. The sub-workflow receives named inputs, validates them, performs the
Google Sheets operation and returns a structured result.

**Trade-offs:** named inputs form a contract, so malformed input can be rejected
rather than written; the rule that incomplete leads are never saved is enforced
structurally. The sub-workflow is reusable and separately testable. Costs: more
components, an environment variable for sub-workflow execution, and a trace that
spans two workflows.

## Final Choice

**Option B.**

- AC11 requires that incomplete leads are never written. Enforcing this inside the
  sub-workflow makes it structural rather than dependent on prompt wording.
- Malformed tool input is one of the required test situations; named inputs make
  it possible to construct and observe.
- Tools are kept as separate sub-workflow JSON files.

## Workflow Diagram

```text
Telegram Trigger
      │
      ▼
   AI Agent ──── Chat Model: Google Gemini (gemini-3.6-flash)
      │     └─── Memory: session key = Telegram chat id
      │     └─── Tool: Call n8n Workflow Tool
      │                     │
      │                     ▼
      │        ┌─────────────────────────────┐
      │        │ Sub-workflow: sheet-tool    │
      │        │ 1. receive named inputs     │
      │        │ 2. validate and normalise   │
      │        │ 3. Google Sheets read/append│
      │        │ 4. return structured result │
      │        └─────────────────────────────┘
      ▼
Telegram: reply to the same chat
```

---

# 3. Mock Data

## Data Model

Google Sheet `n8n-lead-qualification-test`, four tabs.

| Tab | Purpose | Columns |
|---|---|---|
| `services` | Services the agency offers | `service_id`, `name`, `description`, `min_budget_eur`, `typical_timeline` |
| `qualification_rules` | Classification rule | `rule_id`, `qualification`, `condition`, `save_lead`, `next_action` |
| `lead_status_reference` | Status meanings and next actions | `status`, `meaning`, `next_action`, `owner` |
| `leads` | Recorded leads | `lead_id`, `created_at`, `chat_id`, `contact`, `company_type`, `service_requested`, `budget_eur`, `timeframe`, `qualification`, `status`, `next_action`, `uncertainty`, `raw_request` |

### Qualification Rule

| Qualification | Condition | Lead saved |
|---|---|---|
| `incomplete` | Contact missing, service not identifiable, or budget not stated | No — ask one follow-up question |
| `cold` | Service not in the list, or budget below 1000 EUR | Yes |
| `warm` | Service in the list and budget 1000–2499 EUR | Yes |
| `hot` | Service in the list and budget 2500 EUR or above | Yes |

## Edge Cases Included

| Record | Situation | Tests |
|---|---|---|
| L001 | Complete, high value | Normal `hot` path |
| L002 | Budget exactly 2000 | Boundary at the `hot` threshold — must be `warm` |
| L003 | Budget 800 | `cold` by budget |
| L004 | Budget stated approximately | `uncertainty` column |
| L005 | No budget | Missing information |
| L006 | No contact | Lead must not be saved |
| L007 | Service not offered | Unsupported request |
| L008 | Budget "1000, actually 4000"; timeframe "next week or next quarter" | Contradictory information |

## Data Files

| File | Rows |
|---|---|
| `data/services.csv` | 4 |
| `data/qualification_rules.csv` | 4 |
| `data/lead_status_reference.csv` | 4 |
| `data/leads.csv` | 8 |

All records are synthetic test data. Email addresses use the reserved `.example` domain.

## AI Work Log

| Tool or provider | Question asked | Answer used | What I changed | How I verified it |
|---|---|---|---|---|
| AI assistant + n8n docs | Does a Sheets node placed after the Agent satisfy the tool-use requirement? | No — tools must attach to the Agent's own Tool slot as sub-nodes; a node placed after the Agent in the main flow is a fixed step, not something the model chooses to call | Chose the `Call n8n Workflow Tool` + sub-workflow architecture (Option B) over a direct Sheets tool | Calculator attached to the Tool slot appeared in the execution trace during the smoke test (S-B) |
| AI assistant | Why does `gemini-2.5-flash` return a 404 for a new API key? | The model list in n8n's node lags behind provider availability; `gemini-3.6-flash` is current for new keys | Switched the Chat Model node's model | Smoke tests A and B both passed afterward |
| AI assistant | Why does the Google OAuth sign-in window never open from n8n? | Most likely the browser's own popup blocker, not an n8n or Google-side error | Allowed popups for `localhost:5678` | OAuth window opened correctly on the next attempt |
| AI assistant + n8n docs | Why does `Simple Memory` throw "No session ID found" for a Telegram-triggered run? | The node looks for a field literally named `sessionId`, which only `Chat Trigger` happens to output; other triggers must supply the same field name themselves | Added an `Edit Fields` node between `Telegram Trigger` and the Agent, writing `sessionId` (from `message.chat.id`) and `chatInput` (from `message.text`) | Execution trace shows `Simple Memory` succeeding for both a chat-panel session and a live Telegram chat, each keeping separate history |
| AI assistant + n8n/ngrok docs | How to expose local n8n over HTTPS for Telegram webhooks without Docker Desktop's usual setup? | Run ngrok as a second container on the same Docker network, pointed at `n8n:5678`, with a free static domain reserved in the ngrok dashboard | Added the `ngrok` service to `docker-compose.yaml`, set `WEBHOOK_URL` to the ngrok domain, and added a second OAuth redirect URI in Google Cloud alongside the localhost one | `getWebhookInfo` showed the registered URL and `pending_update_count: 0`; a live Telegram message reached the Agent and got a reply in the same chat |
| AI assistant | Does n8n's declared `String` type on a Workflow Input Schema field actually reject a non-string value? | No — it is descriptive only; a JSON object passed through untouched and was serialized as text by the downstream Sheets node | Added a `Code` node that checks `typeof` on every field and throws a clear error for anything that is not a string | Re-running the same malformed input now stops at the `Code` node with the expected error instead of writing garbage to the sheet |
