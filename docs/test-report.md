# Test Report

## Provider and Tool Smoke Test

Workflow: `00 — Smoke test`. Executed before building the domain workflow.

### Provider Record

| Field | Value |
|---|---|
| Provider name | Google (Gemini API via Google AI Studio) |
| Model name | `gemini-3.6-flash` |
| Credential type | Google Gemini(PaLM) API |
| Base URL | `https://generativelanguage.googleapis.com` |
| Date tested | 2026-08-24 |
| Tool-calling result | PASS |
| Structured-output result | Not tested yet |
| Rate or quota limitations | Google AI Studio free tier; no limit reached during testing |

### Smoke Test Results

| ID | Input or setup | Expected behavior | Actual behavior | Result | Evidence |
|---|---|---|---|---|---|
| S-A | `What is the capital of Spain?` | Execution succeeds, readable answer naming Madrid | `The capital of Spain is Madrid.` Success in 1.647 s, 16 tokens | PASS | Screenshot: nodes green, Logs show trigger → AI Agent → Gemini |
| S-B | Calculator attached to the Tool slot. `What is 8347 multiplied by 2913? Use the calculator tool.` | Trace shows the Agent calling Calculator and using the returned value | Trace `AI Agent → Gemini → Calculator → Gemini`. Calculator input `8347 * 2913`, output `24314811`. Reply `8347 multiplied by 2913 is 24,314,811.` Two model calls, 353 tokens | PASS | Screenshot of Logs panel with Calculator input and output |
| S-C | Tool parameter generation | Required fields present, numbers as numbers, missing fields not invented | Agent produced `8347 * 2913` from natural language. Multiple named fields and missing-value handling not yet covered | PARTIAL | Same execution as S-B |
| S-D | Telegram round trip | Trigger receives the message, correct chat id, reply to the same chat, no memory shared between chats | Message sent to the bot via Telegram, agent replied in the same chat listing the four services and asking a follow-up question. Session key normalized to work for both the chat panel and Telegram (see prompt-log) | PASS | Reply received in the live Telegram chat; webhook confirmed registered via `getWebhookInfo` |
| S-E | Google Sheets tool round trip | Trace shows the tool call and the expected sheet result | Not executed | — | — |

## Domain Test Report

| ID | Input or setup | Expected behavior | Actual behavior | Result | Evidence |
|---|---|---|---|---|---|
| T01 | Normal valid request | `We need a Telegram bot answering customer questions. Budget around 3200 euro. Ana ana@testshop.example` | Full request classified and saved | Chain `read_rules → read_services → save_lead`, 3.7 s, 5048 tokens. Row saved: `service_requested=S01`, `qualification=hot`, `budget_eur=around 3200 euro` (client wording kept), `uncertainty=budget stated as approximate` | PASS | Sheet row L12 |
| T02 | Missing required information | `We would like a WhatsApp assistant for our clients. Irina irina@testbuild.example` (no budget stated) | Lead not saved; agent asks one follow-up question about budget | First run exposed a real defect: `qualification_rules.csv` rule R1 only covered a missing contact or unidentifiable service, not a missing budget, so the agent classified the lead `cold` and saved it (row L46), contradicting `AC11`. Fixed R1 to also cover an unstated budget, in both the CSV and the live sheet, and in the `discovery.md` rule table. Re-run: trace stops at `read_rules → read_services`, no `save_lead` call, no new sheet row, reply asks only for budget or timeframe | PASS (after fix) | See `failure-log.md` F09; sheet row L46 is the pre-fix defect evidence, kept for the record |
| T03 | Unknown question or unsupported request | `Can you build a mobile app for our drivers? Budget 5000 euro. Tom tom@testfarm.example` (service not in the agency's list) | Classified `cold` per R2, saved with a decline note, honest reply about what the agency actually does | Chain `read_rules → read_services → save_lead`. Row saved: `budget_eur=5000`, `qualification=cold`, `next_action=Send a polite decline with a referral note`. Reply named the four real services instead of pretending to help with mobile apps | PASS | Sheet row L66 |
| T04 | Contradictory information | `We need CRM integration. Budget 1000. Actually 4000. We need it next week or maybe next quarter. Sara sara@testfin.example` | Conflict is queried, not resolved silently (`AC14`) | Trace stops at `read_rules → read_services`, no `save_lead` call. Reply asked to clarify both the budget (€1,000 or €4,000) and the timeframe (next week or next quarter) in one message | PASS | Execution log, no new sheet row |
| T05 | Information missing from Google Sheets | `What's the starting price for service S05?` (only S01-S04 exist in `services.csv`) | Agent reports the service does not exist instead of inventing a price | Replied "We do not offer a service with the code S05", then listed the four real services with their real `min_budget_eur` values (1500, 2000, 2500, 1000), matching `services.csv` exactly | PASS | Execution log |
| T06 | Wrong Telegram chat ID | `Send a text message` node's Chat ID temporarily set to `1`, a nonexistent chat | Node fails visibly instead of silently succeeding | Telegram API rejected the request: `Bad Request: chat not found`. n8n surfaced it as a clear node error, execution stopped there | PASS | Screenshot of the failed node and error detail |
| T07 | Missing or invalid credential | Not staged separately; a real instance occurred naturally during this project | The system fails visibly and recovers cleanly | The Google Sheets credential's refresh token expired after about ten days of inactivity (Google's Testing-status limit). `Append row in sheet` could not load its document list and named the exact credential needing reconnection. Reconnected via Sign in with Google; the sheet and tab dropdowns loaded normally afterward | PASS | See `failure-log.md` F08 |
| T08 | Model that does not perform tool calling | All three tool connections temporarily removed from the `AI Agent` node's Tool slot, then the same T01-style complete request sent | Agent should not falsely claim success; a clear explanation is preferable to silence | Execution succeeded (Simple Memory + Google Gemini Chat Model only, no tool nodes in the trace, 482 tokens) but returned `{ "output": "" }` — an empty reply. Rule 8 (never confirm success without a real tool result) held: the agent did not lie about saving the lead. It also did not explain the problem to the user, which rule 8 does not cover — noted in Known Limitations | PARTIAL | Execution log, tool connections restored afterward |
| T09 | Malformed tool input | `Tool - Save lead`'s trigger given pinned mock data with `budget_eur` set to an object (`{"amount":5000,"currency":"EUR"}`) instead of a string, run directly against the sub-workflow, bypassing the Agent | Malformed input is rejected with a clear error, not silently written | First run exposed a real gap: n8n's declared `String` type on the Workflow Input Schema is not enforced, and the Sheets node silently serialized the object into the cell as text (row L76). Fixed by adding a `Code` node between the trigger and `Append row in sheet` that checks every field's type and throws `Malformed tool input: field "X" must be a string, got object` on any non-string value. Re-run with the same malformed input: execution stops at the Code node with that error, no write reaches the sheet | PASS (after fix) | See `failure-log.md` F10; row L76 (pre-fix defect evidence) removed from the sheet after recording |
| T10 | Agent attempting to invent information | `Looking to connect our CRM with email. We can spend 2000 euro. Marc marc@testlogix.example` (no company type stated) | `company_type` left empty, not invented | Row saved with `company_type` blank, `qualification=warm` (2000 EUR boundary). System prompt rule against inventing missing fields held | PASS | Sheet row L20; earlier run with the D01 defect (before the `contact` field description fix) is row L16 |
