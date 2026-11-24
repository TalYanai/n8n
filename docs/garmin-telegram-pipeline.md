# Garmin CSV via Telegram - n8n front door to an analyzer service

Evidence: **A** = verified in the exported file, **B** = inferred or taken from the project's own README, not checked.

## Context and authorship
- Part of a larger personal project (the stack also includes a Python analyzer service, a database and a dashboard). The n8n workflow is only the intake and reply layer. The analysis and database logic live in a separate service; this workflow is not "the AI".
- I designed and specified the flow and assembled it with n8n's visual editor and AI coding assistants. I did not examine the analyzer's code for this write-up and claim nothing about it beyond the project README (B).

## Purpose
Drop a Garmin Connect activity export (CSV) into a Telegram chat, have it analysed and stored, and get a Hebrew summary back in the same chat. Downstream the stored data feeds a Power BI dashboard (README).

## Trigger
Telegram Trigger node (webhook mode; n8n registers the bot webhook when the workflow is activated - stated in the node's own note, A). Listens for incoming messages.

## Node-by-node flow (8 nodes, A)
| # | Node (plain English) | Type | What it does |
|---|---|---|---|
| 1 | Telegram trigger | Telegram Trigger | Receives a chat message |
| 2 | IF: is it a CSV? | IF | True if the attached document's file name ends in `.csv` (case-insensitive) |
| 3 | HTTP: get file info | HTTP Request | Asks the Telegram Bot API for the file path of the attachment |
| 4 | HTTP: download CSV | HTTP Request | Downloads the file as plain text (Garmin files are UTF-8, sometimes with a BOM; the Python side handles it) |
| 5 | HTTP: analyze and save | HTTP Request | POSTs the CSV text as JSON to the internal analyzer service on the Docker network; 120-second timeout for large CSVs plus model latency |
| 6 | Code: build message | Code (JavaScript) | Builds a Hebrew summary: new rows inserted, activities in file, total distance, average heart rate, calories, breakdown by activity type, and a "no duplicates added" note when nothing new was inserted |
| 7 | Telegram: confirm | Telegram | Sends the summary back to the same chat (chat id taken from the trigger by expression) |
| 8 | Telegram: reject | Telegram | Replies in Hebrew asking for a CSV export from Garmin Connect only |

Diagram: [garmin-telegram-pipeline.svg](garmin-telegram-pipeline.svg).

## Branching and error handling
- One branch: non-CSV messages go to the reject reply (A).
- **No error branch, no error workflow, no retries** on the three HTTP nodes (A). If the analyzer or Telegram API fails, the user gets no reply and the failure is only visible in n8n's execution list.
- Duplicate handling is not in n8n; per the README the database upsert ignores duplicates and the Code node only reports it.

## AI nodes
**None in n8n (A).** Per the README (B): the analyzer service calls the Claude API with a tool-use schema so the output is schema-valid JSON, normalising units (miles to km, feet to metres, HH:MM:SS to minutes), then upserts to PostgreSQL.

## Integrations (by type)
Telegram Bot credential (trigger, confirm, reject); Telegram Bot HTTP API (get file, download); an internal HTTP analyzer service on the Docker network. Downstream per the README: PostgreSQL, Power BI.

## Contribution split
- **Me (A for the workflow file; B for who built what):** the whole workflow - Hebrew messages, node notes and the summary builder are project-specific. It is not a template.
- **Third party:** standard n8n and Telegram nodes only.

## Known limits
1. No error handling or alerting (see above); no input size guard beyond the file-type check.
2. The bot token is placed in request URLs via an expression from an environment variable, so it may appear in n8n execution data and logs; a credential-based Telegram node setup would be tidier (B).
3. No check that the sender is the bot owner: anyone who finds the bot could send a CSV (B).
4. The CSV is sent whole in a single JSON body, with no chunking; large exports rely on the 120 s timeout.
5. A manual step remains: exporting the CSV from Garmin Connect.
6. Not executed for this review; the workflow's `active` state and run history are not in the file.
