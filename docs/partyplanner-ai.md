# PartyPlanner AI - multi-agent party invitation automation

Evidence levels: **A** = verified in the exported workflow file, **B** = inferred or a static reading. Nothing was executed; the exported files were only read.

## Context and authorship
- Academic **group project**. Teammates built the web front end that collects the party details. The automation and agent layer described here is my part. The files show only the n8n side, so the team split is **stated by me (B)**, not provable from the files.
- What the files do show (A): 13 exported versions made over 8-14 June 2026, a real iteration history of one workflow.
- My part is design and specification: the two-agent split, the prompts, which tools each agent gets, input validation, error alerting and iterating through versions. The workflow was assembled with n8n's visual editor and AI coding assistants. The front end, the invitation image template and the data model are not mine.

## Purpose
A front end posts the details of a planned party (name, date, guest e-mails, shopping quantities, task checklist, organizer e-mail). The workflow turns this into (1) a designed invitation image e-mailed to every guest and (2) a separate e-mail to the organizer only, with a shopping-quantities table and a task checklist.

## Trigger
HTTP POST webhook (n8n Webhook node, response mode "respond via node"). The caller is the team's web front end (B).

## Node-by-node flow (final version)
19 nodes in the file = **17 functional nodes + 2 sticky notes** (A). Original labels mix Hebrew and English; translated here.

| # | Node (plain English) | Type | What it does |
|---|---|---|---|
| 1 | Webhook - party ready | Webhook | Receives the POST payload |
| 2 | Code - clean inputs | Code (JavaScript) | Reads the body; strips unsafe characters from e-mail strings; joins the guest list; renders quantities and checklist as bullet text; computes guest count |
| 3 | IF - validate input | IF | Checks the party name and organizer e-mail are not empty |
| 4 | Agent 1 - Designer | AI Agent (LangChain) | Writes festive invitation wording as JSON |
| 5 | LLM - Designer | Anthropic chat model | Claude Sonnet 4.5, temperature 0.7 |
| 6 | Render image | Image-template service node | Fills the name, date and guest-count layers of a pre-made invitation template and returns an image link |
| 7 | Set - extract image link | Set | Keeps the image link and carries the other fields forward |
| 8 | Agent 2 - Sender | AI Agent | Sends the two e-mails by calling tools |
| 9 | LLM - Sender | Anthropic chat model | Claude Sonnet 4.5, temperature 0 |
| 10 | Tool: send_invitations | Gmail Tool | Lets the agent send the invitation to all guests |
| 11 | Tool: send_quantities | Gmail Tool | Lets the agent send the shopping list and checklist to the organizer only |
| 12 | Respond 200 - success | Respond to Webhook | Returns JSON `status: done` plus the agent's final text |
| 13 | If - has organizer mail | IF | On the validation-failed branch: is there someone to notify? |
| 14 | Send a message | Gmail | Hebrew e-mail to the organizer: data missing, please resubmit the form |
| 15 | Respond 400 - validation failed | Respond to Webhook | Returns HTTP 400 "missing required fields" |
| 16 | Error Trigger | Error Trigger | Starts on a workflow error |
| 17 | Error handler - Gmail admin | Gmail | Alerts an admin address (read from an environment variable) with execution id, error message and time |
| - | 2 sticky notes | Sticky Note | Section notes (intake and validation; sender agent) |

Diagram: [partyplanner-ai.svg](partyplanner-ai.svg).

## Branching and error handling
- **Validation branch (A):** IF node. True goes to the Designer path. False goes to "has organizer mail?"; true there e-mails the organizer asking to resubmit, false returns HTTP 400.
- **Cost guard (A, from a sticky note):** validation exists so model tokens are not spent on empty requests.
- **Defensive input cleaning (A):** the Code node and the Gmail tool expressions strip characters outside a safe e-mail character set, so a caller or the model cannot slip odd characters into the To field.
- **Silent-failure protection (A, design intent):** an Error Trigger plus a Gmail alert node exists (see limitation 4).

## AI nodes
| Agent | Model | Role and system-prompt gist | Tools | Output |
|---|---|---|---|---|
| Designer | Claude Sonnet 4.5, temp 0.7 | Party-invitation designer; return only JSON with a festive name, a formatted date and the guest count. Hebrew prompt. | none in the final version | JSON object; no output-parser node attached |
| Sender | Claude Sonnet 4.5, temp 0 | Invitation-sending system; do two steps in exact order with the tools; write right-to-left HTML e-mail bodies; guests get the image, the organizer gets a quantities table and checklist; answer only "sent successfully". Hebrew prompt. | `send_invitations`, `send_quantities` (Gmail tools; the model fills recipient, subject and body through n8n's "from AI" fields) | Short confirmation string returned in the 200 response |

Maximum 5 agent iterations each. No memory in the final version (an earlier version had a window-buffer memory per agent; it was removed).

## Integrations (by type)
Anthropic API credential; Gmail OAuth credential (tools, alert and fallback e-mail); image-rendering API credential; inbound HTTP webhook.

## Contribution split
- **Me (stated, B; consistent with the file history, A):** agent layer design - prompts, two-agent design, tool selection, validation, error alerting, version iteration.
- **Team:** the web front end that produces the payload (B; not in the files).
- **Third party:** the invitation image template lives in the image-service account and is not in the export.

## How it evolved (13 versions, 8-14 June 2026)
Ordered by file timestamp. The seven earliest files have no workflow id or version id and no active flag, which suggests drafts that were generated or assembled and then imported; the later files carry full n8n export metadata (B).

| Step | Change (A, from a structural diff) |
|---|---|
| 1 | **Single agent**, 11 nodes. One prompt orders 3 steps: build the invitation (HTTP-request tool to an image API), e-mail guests, e-mail the organizer. Validation required non-empty guests and quantities. |
| 2 | Added the Error Trigger node (12 nodes) |
| 3 | Separate LLM node (Claude Sonnet 4.6) and a Simple Memory node (13 nodes) |
| 4 | **Split into two agents**: Designer (temp 0.7) and Sender (temp 0), each with memory; a step to extract the image link between them (17 nodes) |
| 5 | Memory removed; Gmail tool nodes used for sending; models switched to Sonnet 4.5; prompt tweaks (15 nodes) |
| 6 | First real n8n exports; the HTTP image step replaced by the native image-template node |
| 7 | 19 nodes: sticky-note documentation, `send_invitations` and `send_quantities` Gmail tools, new fallback branch (organizer e-mail check, "data missing" e-mail) |
| 8 | Image rendering moved from an agent tool to a plain node after the Designer (deterministic image step); prompt and tool-description tuning around e-mail address handling |
| 9 | Latest version (marked active): Set node replaced by a **Code node** with input sanitising and text formatting; validation changed from "guests + quantities" to "name + organizer e-mail"; workflow renamed |

The story: one prompt-driven agent that did everything, then two single-purpose agents, then the deterministic image-rendering step moved out of the model into an ordinary node.

## Known limits (static reading of the export; B unless stated)
1. **Designer output appears unused (A on the wiring):** the image and Set nodes read name, date and guest count from the Code node, not from the Designer's JSON. As wired, the Designer adds model cost but its festive text does not reach the image. It has no tools and no output parser.
2. **Validation is narrower than documented:** the party-name condition carries a trailing newline in its expression, so it is likely never "empty"; effectively only the organizer e-mail is checked. The sticky note says guests and quantities are validated, which no longer matches (A on the mismatch; B on the effect).
3. **Fallback branch looks fragile:** the organizer-mail check and the fallback e-mail expressions start with a doubled equals sign, which probably yields a string beginning with "=", so the "no organizer" test may always pass and the 400 response may be unreachable. The fallback e-mail node has no outgoing connection, so the web caller gets no response on that path (A for the missing connection; B for the rest). Some prompt instructions and regex clean-ups read like workarounds for the same symptom.
4. **Error handler is probably inert:** the Error Trigger sits inside the same workflow and no error workflow is set in the settings; an Error Trigger normally belongs to a separate workflow (B).
5. No automated tests and no execution logs in the files: **it is not proven that it ran end to end.** The final file is marked active (A), which only shows it was switched on.
6. Model choice dropped from Sonnet 4.6 to Sonnet 4.5 in later versions; reason not recorded.
7. The Gmail tools send mail on a model's decision; there is no human-approval step or rate limit.
